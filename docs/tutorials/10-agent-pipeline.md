# Tutorial 10: Building an Agent Pipeline

**Prerequisites:** Tutorial 9 (multi-agent dispatcher). You should understand `dispatchTo`,
awaited subprocesses, and system prompt injection before this tutorial.

**Navigation:** [← Tutorial 9](./09-multi-agent-dispatcher.md)

---

## What You Will Build

A `review-pipeline` extension implementing the **pipeline pattern**: a fixed 3-step
sequence where each step's output feeds into the next step's input.

```
User request
    ↓
 Scout      (explores the codebase, reports what it found)
    ↓
 Builder    (implements changes based on scout's report)
    ↓
 Reviewer   (reviews the builder's work and reports issues)
    ↓
Final result to primary agent
```

The primary agent has a `run_pipeline` tool and full access to regular tools. When the
work warrants a full pipeline run, it calls `run_pipeline`. For simple questions it
can answer directly.

A progress widget shows all three step cards with live status:

```
┌──────────────┐  ──▶  ┌──────────────┐  ──▶  ┌──────────────┐
│ Scout        │       │ Builder      │       │ Reviewer     │
│ ✓ done 14s  │       │ ● running 7s │       │ ○ pending    │
│ Found 3 file │       │ Writing fun… │       │ —            │
└──────────────┘       └──────────────┘       └──────────────┘
```

---

## Background: Pipeline vs. Dispatcher

### How They Differ

| Property | Dispatcher (Tutorial 9) | Pipeline (Tutorial 10) |
|---|---|---|
| Step order | Dynamic — chosen by LLM | Fixed — always scout → builder → reviewer |
| Data flow | Each dispatch is independent | Output of step N is the input to step N+1 |
| Agent selection | LLM picks the right agent | Hardcoded sequence |
| Primary agent tools | Only `dispatch_agent` | Full tool set + `run_pipeline` |
| Use case | LLM decides what to delegate | User invokes a repeatable workflow |

### When to Use Each

**Use a pipeline when:** The workflow is sequential and repeatable. Plan first, then
build, then review is almost always the right order. You want consistent quality control
on every significant change.

**Use a dispatcher when:** The coordinator needs to think about which specialist is
appropriate for the task. The answer changes based on what the user is asking for.

**Use parallel execution when:** Multiple independent questions can be answered
simultaneously. Tutorial 8's background subagents are one form of this — several
research questions running concurrently.

### Variable Substitution

The key mechanism that makes pipelines work is variable substitution in step prompts:

```
step 1 prompt: "Explore and report on: $INPUT"
step 2 prompt: "Based on this analysis:\n$INPUT\n\nImplement: $ORIGINAL"
step 3 prompt: "Review this implementation:\n$INPUT\n\nOriginal request: $ORIGINAL"
```

When step 2 runs, `$INPUT` is replaced with step 1's output. `$ORIGINAL` is always the
user's original task — it gives every step access to the full context even though each
step only receives the previous step's output directly.

This is how `agent-chain.ts` works in production, and it is what makes pipelines more
powerful than simply running three independent agents.

---

## What You Will Learn

- The pipeline/chain pattern: sequential execution with output chaining
- `$INPUT` and `$ORIGINAL` variable substitution in step prompts
- Sequential `Promise` execution using `for...of` with `await`
- Step state tracking: `pending → running → done → error`
- Rendering step cards with arrows connecting them
- How the primary agent keeps its tools while the pipeline does the heavy work
- When to call `run_pipeline` vs. working directly

---

## Step 1: Define the Pipeline Steps

```typescript
/**
 * Review Pipeline — run_pipeline tool executes a 3-step Scout → Build → Review workflow
 *
 * Each step's output becomes the next step's $INPUT.
 * $ORIGINAL is always the user's original task.
 *
 * Usage: pi -e extensions/review-pipeline.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";
const { spawn } = require("child_process") as typeof import("child_process");
import * as fs from "fs";
import * as path from "path";
```

Define the step structure. Each step is a self-contained description of what one
specialist should do:

```typescript
interface PipelineStep {
  name: string;
  prompt: string;    // template with $INPUT and $ORIGINAL placeholders
  tools: string;
  systemPrompt: string;
}

const STEPS: PipelineStep[] = [
  {
    name: "scout",
    prompt: "Explore the codebase and gather all information needed to accomplish this task: $INPUT",
    tools: "read,grep,find,ls",
    systemPrompt: `You are a scout agent in a pipeline. Your job is to explore
the codebase and produce a comprehensive report of what you found. Focus on:
file locations, existing patterns, relevant code sections, and what would need
to change to accomplish the task. Be specific — the builder who comes after
you needs enough detail to implement without re-reading everything. Do NOT
modify any files.`,
  },
  {
    name: "builder",
    prompt: `Based on this reconnaissance report, implement the task.

SCOUT REPORT:
$INPUT

ORIGINAL TASK:
$ORIGINAL`,
    tools: "read,write,edit,bash,grep,find,ls",
    systemPrompt: `You are a builder agent in a pipeline. You receive a scout
report and implement the required changes. Follow existing code patterns.
Write complete, working code. After implementing, produce a summary of:
what files you changed, what you added or removed, and how to test the change.`,
  },
  {
    name: "reviewer",
    prompt: `Review the implementation described below.

IMPLEMENTATION SUMMARY:
$INPUT

ORIGINAL TASK:
$ORIGINAL

Read the relevant files and report any issues you find.`,
    tools: "read,bash,grep,find,ls",
    systemPrompt: `You are a reviewer agent in a pipeline. You receive an
implementation summary and review the actual code. Check for: correctness
(does it do what was asked?), style (does it match the project's patterns?),
bugs (edge cases, off-by-one errors, error handling), and security (no
injection, no hardcoded secrets). Report each issue with a severity rating
(Critical, High, Medium, Low) and a concrete suggestion. Do NOT modify files.`,
  },
];
```

**Why is the scout's prompt simpler than the builder's?** Because scout is the first
step: it has only `$INPUT` (the original task). Builder is step 2, so it receives both
`$INPUT` (scout's report) and `$ORIGINAL` (the user's task). Using both gives builder
full context without losing the user's original intent in the chain of handoffs.

---

## Step 2: Define Step State

```typescript
interface StepState {
  name: string;
  status: "pending" | "running" | "done" | "error";
  elapsed: number;
  lastWork: string; // last non-empty output line from this step's agent
}
```

`pending` means the step has not started yet. `running` means the subprocess is active.
`done` or `error` means it finished. This enum drives the widget's visual state.

---

## Step 3: Write the Extension Shell

```typescript
export default function (pi: ExtensionAPI) {
  let stepStates: StepState[] = resetStepStates();
  let widgetCtx: any = null;
  let sessionDir = "";

  function resetStepStates(): StepState[] {
    return STEPS.map((s) => ({
      name: s.name,
      status: "pending" as const,
      elapsed: 0,
      lastWork: "",
    }));
  }
```

`"pending" as const` is a TypeScript type assertion. Without it, TypeScript infers the
type of the string literal as `string`. The `as const` narrows it to the literal type
`"pending"`, which is required to satisfy the union type `"pending" | "running" | "done" | "error"`.
This is a common TypeScript pattern when building objects from literals that must match
a union.

---

## Step 4: Write the Widget

The pipeline widget renders step cards connected by arrows. The number of steps is
fixed at three, so the layout is straightforward:

```typescript
  function updateWidget() {
    if (!widgetCtx) return;

    widgetCtx.ui.setWidget("pipeline", (_tui: any, theme: any) => {
      return {
        render(width: number): string[] {
          const arrowWidth = 5;      // " ──▶ "
          const cols = STEPS.length; // 3
          // Divide available width equally, minus the arrows between steps
          const totalArrowWidth = arrowWidth * (cols - 1);
          const colWidth = Math.max(12, Math.floor((width - totalArrowWidth) / cols));

          // Render each step as a card (array of lines)
          const cards = stepStates.map((s) => renderCard(s, colWidth, theme));

          // Interleave cards with arrows
          const cardHeight = cards[0].length;
          const arrowRow = 2; // which row gets the arrow (middle of the card)
          const outputLines: string[] = [];

          for (let line = 0; line < cardHeight; line++) {
            let row = cards[0][line];
            for (let c = 1; c < cols; c++) {
              if (line === arrowRow) {
                row += theme.fg("dim", " ──▶ ");
              } else {
                row += " ".repeat(arrowWidth);
              }
              row += cards[c][line];
            }
            outputLines.push(row);
          }

          return outputLines;
        },
        invalidate() {},
      };
    });
  }

  function renderCard(state: StepState, colWidth: number, theme: any): string[] {
    const w = colWidth - 2; // inner width (subtract 2 for borders)
    const truncate = (s: string, max: number) =>
      s.length > max ? s.slice(0, max - 3) + "..." : s;

    const statusColor =
      state.status === "pending" ? "dim" :
      state.status === "running" ? "accent" :
      state.status === "done" ? "success" : "error";

    const statusIcon =
      state.status === "pending" ? "○" :
      state.status === "running" ? "●" :
      state.status === "done" ? "✓" : "✗";

    const displayName = state.name.charAt(0).toUpperCase() + state.name.slice(1);
    const nameStr = theme.fg("accent", theme.bold(truncate(displayName, w)));

    const elapsedStr = state.status !== "pending"
      ? ` ${Math.round(state.elapsed / 1000)}s` : "";
    const statusStr = `${statusIcon} ${state.status}${elapsedStr}`;
    const statusLine = theme.fg(statusColor, statusStr);

    const workText = state.lastWork
      ? truncate(state.lastWork, Math.min(50, w - 1))
      : "";
    const workLine = workText
      ? theme.fg("muted", workText)
      : theme.fg("dim", "—");

    // Build a box: top border, name, status, work preview, bottom border
    const top = "┌" + "─".repeat(w) + "┐";
    const bot = "└" + "─".repeat(w) + "┘";
    const border = (content: string, visLen: number) =>
      theme.fg("dim", "│") +
      content +
      " ".repeat(Math.max(0, w - visLen)) +
      theme.fg("dim", "│");

    return [
      theme.fg("dim", top),
      border(" " + nameStr, 1 + Math.min(displayName.length, w)),
      border(" " + statusLine, 1 + statusStr.length),
      border(" " + workLine, 1 + Math.min(workText.length || 1, w - 1)),
      theme.fg("dim", bot),
    ];
  }
```

The `border()` helper builds one line of the card box. It takes a content string and
its *visible* length (excluding ANSI escape codes, which add bytes but no visual width).
This is the challenge with colored terminal output: the escape codes inflate the string
length, so you cannot use `content.length` to calculate padding. Always track visible
length separately.

---

## Step 5: Write the Single-Step Runner

Like `dispatchTo` in Tutorial 9, this returns a `Promise` that resolves when the
subprocess exits:

```typescript
  function runStep(
    step: PipelineStep,
    stepIndex: number,
    prompt: string,
    ctx: any,
  ): Promise<{ output: string; exitCode: number; elapsed: number }> {
    const state = stepStates[stepIndex];
    state.status = "running";
    state.lastWork = "";
    state.elapsed = 0;
    updateWidget();

    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    const agentSessionFile = path.join(sessionDir, `pipeline-${step.name}.json`);

    // Check if this agent has a session file from a previous pipeline run
    const hasSession = fs.existsSync(agentSessionFile);

    const args = [
      "--mode", "json", "-p", "--no-extensions",
      "--model", model,
      "--tools", step.tools,
      "--thinking", "off",
      "--append-system-prompt", step.systemPrompt,
      "--session", agentSessionFile,
    ];

    // Continue the session if one exists (agent remembers previous pipeline runs)
    if (hasSession) args.push("-c");

    args.push(prompt);

    const textChunks: string[] = [];
    const startTime = Date.now();

    const timer = setInterval(() => {
      state.elapsed = Date.now() - startTime;
      updateWidget();
    }, 1000);

    return new Promise((resolve) => {
      const proc = spawn("pi", args, {
        stdio: ["ignore", "pipe", "pipe"],
        env: { ...process.env },
      });

      let buffer = "";

      proc.stdout!.setEncoding("utf-8");
      proc.stdout!.on("data", (chunk: string) => {
        buffer += chunk;
        const lines = buffer.split("\n");
        buffer = lines.pop() || "";
        for (const line of lines) {
          if (!line.trim()) continue;
          try {
            const event = JSON.parse(line);
            if (event.type === "message_update") {
              const delta = event.assistantMessageEvent;
              if (delta?.type === "text_delta" && delta.delta) {
                textChunks.push(delta.delta);
                const full = textChunks.join("");
                state.lastWork =
                  full.split("\n").filter((l: string) => l.trim()).pop() || "";
                updateWidget();
              }
            }
          } catch {}
        }
      });

      proc.stderr!.setEncoding("utf-8");
      proc.stderr!.on("data", () => {});

      proc.on("close", (code) => {
        if (buffer.trim()) {
          try {
            const ev = JSON.parse(buffer);
            if (ev.type === "message_update" &&
                ev.assistantMessageEvent?.type === "text_delta") {
              textChunks.push(ev.assistantMessageEvent.delta || "");
            }
          } catch {}
        }
        clearInterval(timer);
        state.elapsed = Date.now() - startTime;
        state.status = code === 0 ? "done" : "error";
        const output = textChunks.join("");
        state.lastWork =
          output.split("\n").filter((l: string) => l.trim()).pop() || "";
        updateWidget();
        resolve({ output, exitCode: code ?? 1, elapsed: state.elapsed });
      });

      proc.on("error", (err) => {
        clearInterval(timer);
        state.status = "error";
        state.lastWork = `Error: ${err.message}`;
        updateWidget();
        resolve({
          output: `Error running ${step.name}: ${err.message}`,
          exitCode: 1,
          elapsed: Date.now() - startTime,
        });
      });
    });
  }
```

---

## Step 6: Write the Pipeline Orchestrator

This is the function that runs all steps sequentially. The critical pattern is the
`for...of` loop with `await` inside:

```typescript
  async function runPipeline(
    task: string,
    ctx: any,
  ): Promise<{ output: string; success: boolean; elapsed: number }> {
    const pipelineStart = Date.now();

    // Reset all steps to pending before each pipeline run
    stepStates = resetStepStates();
    updateWidget();

    let input = task;               // $INPUT for step 1 is the original task
    const originalTask = task;      // $ORIGINAL is always the user's task

    // for...of with await runs steps sequentially.
    // The loop body does not advance until the current step's Promise resolves.
    for (let i = 0; i < STEPS.length; i++) {
      const step = STEPS[i];
      stepStates[i].status = "running";
      updateWidget();

      // Replace $INPUT with previous step's output, $ORIGINAL with user's task
      const resolvedPrompt = step.prompt
        .replace(/\$INPUT/g, input)
        .replace(/\$ORIGINAL/g, originalTask);

      const result = await runStep(step, i, resolvedPrompt, ctx);

      if (result.exitCode !== 0) {
        // Pipeline fails fast: if any step errors, stop the chain
        stepStates[i].status = "error";
        updateWidget();
        return {
          output: `Pipeline failed at step ${i + 1} (${step.name}): ${result.output}`,
          success: false,
          elapsed: Date.now() - pipelineStart,
        };
      }

      stepStates[i].status = "done";
      updateWidget();

      // The output of this step becomes the input of the next step
      input = result.output;
    }

    return {
      output: input, // final step's output
      success: true,
      elapsed: Date.now() - pipelineStart,
    };
  }
```

**The `for...of` + `await` pattern is the heart of sequential pipelines.** In contrast:
- `array.forEach()` does not work with `await` — callbacks run in parallel.
- `Promise.all([p1, p2, p3])` runs all promises concurrently — also parallel.
- Only `for...of` (or a traditional `for` loop) with `await` inside guarantees sequential
  execution where each iteration waits for the previous to complete.

**The `$INPUT` replacement chain:**

```
task = "add logging to the server"
Step 1: prompt = "Explore... $INPUT"
        → resolved: "Explore... add logging to the server"
        → output: "Found server.ts at src/server.ts, logger module..."

Step 2: prompt = "Based on scout report:\n$INPUT\nImplement: $ORIGINAL"
        → resolved: "Based on scout report:\nFound server.ts...\nImplement: add logging..."
        → output: "Added winston logger to server.ts, line 42..."

Step 3: prompt = "Review implementation:\n$INPUT\nOriginal: $ORIGINAL"
        → resolved: "Review implementation:\nAdded winston...\nOriginal: add logging..."
        → output: "Implementation looks correct. One medium issue: missing log rotation..."
```

Each step is fully informed: it knows what it is doing, what came before, and what
the original goal was.

---

## Step 7: Register the run_pipeline Tool and Wire Up Events

```typescript
  pi.registerTool({
    name: "run_pipeline",
    description: "Execute the Scout → Builder → Reviewer pipeline on a task. Use this for significant work: new features, refactors, multi-file changes. For simple questions or quick lookups, answer directly.",
    parameters: Type.Object({
      task: Type.String({
        description: "The task description to pass through the pipeline",
      }),
    }),
    execute: async (_callId, args, _signal, _onUpdate, ctx) => {
      const { task } = args as { task: string };

      const result = await runPipeline(task, ctx);

      const truncated = result.output.length > 8000
        ? result.output.slice(0, 8000) + "\n\n... [truncated]"
        : result.output;

      const status = result.success ? "done" : "error";
      const elapsedSec = Math.round(result.elapsed / 1000);

      return {
        content: [{
          type: "text",
          text: `Pipeline ${status} in ${elapsedSec}s.\n\n${truncated}`,
        }],
      };
    },
  });

  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;

    sessionDir = path.join(ctx.cwd, ".pi", "agent-sessions");
    fs.mkdirSync(sessionDir, { recursive: true });

    // Clean old pipeline session files so agents start fresh each outer session
    try {
      for (const file of fs.readdirSync(sessionDir)) {
        if (file.startsWith("pipeline-") && file.endsWith(".json")) {
          fs.unlinkSync(path.join(sessionDir, file));
        }
      }
    } catch {}

    stepStates = resetStepStates();
    updateWidget();

    ctx.ui.setStatus("pipeline", "Pipeline: Scout → Builder → Reviewer");
    ctx.ui.notify(
      "Review Pipeline loaded.\n" +
      "Call run_pipeline for significant work, or work directly for simple tasks.\n" +
      "Pipeline: Scout → Builder → Reviewer",
      "info"
    );
  });

  // Before each LLM call, inject pipeline description into the system prompt.
  // The primary agent keeps all default tools — it can work directly OR invoke run_pipeline.
  pi.on("before_agent_start", async (event, _ctx) => {
    const stepDescriptions = STEPS.map((s, i) =>
      `${i + 1}. **${s.name.charAt(0).toUpperCase() + s.name.slice(1)}** — Tools: ${s.tools}`
    ).join("\n");

    return {
      systemPrompt: event.systemPrompt + `\n\n## Review Pipeline

You have access to a run_pipeline tool that executes a three-step workflow:

${stepDescriptions}

**When to use run_pipeline:**
- Adding new features or making significant changes
- Tasks that touch multiple files
- Any work you want reviewed before completing
- Refactoring or restructuring

**When to work directly:**
- Simple questions (no files needed)
- Quick file reads or information lookups
- Single-line edits that are obviously correct
- Explaining or summarizing content

The pipeline is sequential: scout's output becomes builder's input, builder's
output becomes reviewer's input. $ORIGINAL (the user's task) is available to
all steps for context.`,
    };
  });
} // end of export default function
```

**Note the difference from Tutorial 9:** Here the primary agent keeps all its default
tools. The `before_agent_start` concatenates with `event.systemPrompt` (augment pattern)
rather than replacing it. The primary agent is a hybrid: it can work directly for simple
things, and call `run_pipeline` for complex things. This is more practical than the pure
dispatcher of Tutorial 9 for everyday use.

---

## Full Final Code

```typescript
/**
 * Review Pipeline — run_pipeline tool executes Scout → Builder → Reviewer
 *
 * Usage: pi -e extensions/review-pipeline.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";
const { spawn } = require("child_process") as typeof import("child_process");
import * as fs from "fs";
import * as path from "path";

interface PipelineStep {
  name: string;
  prompt: string;
  tools: string;
  systemPrompt: string;
}

interface StepState {
  name: string;
  status: "pending" | "running" | "done" | "error";
  elapsed: number;
  lastWork: string;
}

const STEPS: PipelineStep[] = [
  {
    name: "scout",
    prompt: "Explore the codebase and gather all information needed to accomplish this task: $INPUT",
    tools: "read,grep,find,ls",
    systemPrompt: `You are a scout agent in a pipeline. Explore the codebase and produce
a comprehensive report: file locations, existing patterns, relevant code sections, and
what would need to change. Be specific. The builder needs enough detail to implement
without re-reading everything. Do NOT modify any files.`,
  },
  {
    name: "builder",
    prompt: `Based on this reconnaissance report, implement the task.

SCOUT REPORT:
$INPUT

ORIGINAL TASK:
$ORIGINAL`,
    tools: "read,write,edit,bash,grep,find,ls",
    systemPrompt: `You are a builder agent in a pipeline. You receive a scout report and
implement the required changes. Follow existing code patterns. Write complete, working
code. After implementing, produce a summary of: what files you changed, what you added
or removed, and how to test the change.`,
  },
  {
    name: "reviewer",
    prompt: `Review the implementation described below.

IMPLEMENTATION SUMMARY:
$INPUT

ORIGINAL TASK:
$ORIGINAL

Read the relevant files and report any issues you find.`,
    tools: "read,bash,grep,find,ls",
    systemPrompt: `You are a reviewer agent in a pipeline. Review the implementation for:
correctness, style, bugs, and security. Report each issue with severity (Critical, High,
Medium, Low) and a concrete suggestion. Do NOT modify files.`,
  },
];

export default function (pi: ExtensionAPI) {
  let stepStates: StepState[] = resetStepStates();
  let widgetCtx: any = null;
  let sessionDir = "";

  function resetStepStates(): StepState[] {
    return STEPS.map((s) => ({
      name: s.name,
      status: "pending" as const,
      elapsed: 0,
      lastWork: "",
    }));
  }

  function renderCard(state: StepState, colWidth: number, theme: any): string[] {
    const w = colWidth - 2;
    const truncate = (s: string, max: number) =>
      s.length > max ? s.slice(0, max - 3) + "..." : s;
    const statusColor =
      state.status === "pending" ? "dim" :
      state.status === "running" ? "accent" :
      state.status === "done" ? "success" : "error";
    const statusIcon =
      state.status === "pending" ? "○" :
      state.status === "running" ? "●" :
      state.status === "done" ? "✓" : "✗";
    const displayName = state.name.charAt(0).toUpperCase() + state.name.slice(1);
    const nameStr = theme.fg("accent", theme.bold(truncate(displayName, w)));
    const elapsedStr = state.status !== "pending" ? ` ${Math.round(state.elapsed / 1000)}s` : "";
    const statusStr = `${statusIcon} ${state.status}${elapsedStr}`;
    const statusLine = theme.fg(statusColor, statusStr);
    const workText = state.lastWork ? truncate(state.lastWork, Math.min(50, w - 1)) : "";
    const workLine = workText ? theme.fg("muted", workText) : theme.fg("dim", "—");
    const top = "┌" + "─".repeat(w) + "┐";
    const bot = "└" + "─".repeat(w) + "┘";
    const border = (content: string, visLen: number) =>
      theme.fg("dim", "│") + content + " ".repeat(Math.max(0, w - visLen)) + theme.fg("dim", "│");
    return [
      theme.fg("dim", top),
      border(" " + nameStr, 1 + Math.min(displayName.length, w)),
      border(" " + statusLine, 1 + statusStr.length),
      border(" " + workLine, 1 + Math.min(workText.length || 1, w - 1)),
      theme.fg("dim", bot),
    ];
  }

  function updateWidget() {
    if (!widgetCtx) return;
    widgetCtx.ui.setWidget("pipeline", (_tui: any, theme: any) => {
      return {
        render(width: number): string[] {
          const arrowWidth = 5;
          const cols = STEPS.length;
          const totalArrowWidth = arrowWidth * (cols - 1);
          const colWidth = Math.max(12, Math.floor((width - totalArrowWidth) / cols));
          const cards = stepStates.map((s) => renderCard(s, colWidth, theme));
          const cardHeight = cards[0].length;
          const arrowRow = 2;
          const outputLines: string[] = [];
          for (let line = 0; line < cardHeight; line++) {
            let row = cards[0][line];
            for (let c = 1; c < cols; c++) {
              row += line === arrowRow ? theme.fg("dim", " ──▶ ") : " ".repeat(arrowWidth);
              row += cards[c][line];
            }
            outputLines.push(row);
          }
          return outputLines;
        },
        invalidate() {},
      };
    });
  }

  function runStep(
    step: PipelineStep,
    stepIndex: number,
    prompt: string,
    ctx: any,
  ): Promise<{ output: string; exitCode: number; elapsed: number }> {
    const state = stepStates[stepIndex];
    state.status = "running";
    state.lastWork = "";
    state.elapsed = 0;
    updateWidget();

    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    const agentSessionFile = path.join(sessionDir, `pipeline-${step.name}.json`);
    const hasSession = fs.existsSync(agentSessionFile);

    const args = [
      "--mode", "json", "-p", "--no-extensions",
      "--model", model,
      "--tools", step.tools,
      "--thinking", "off",
      "--append-system-prompt", step.systemPrompt,
      "--session", agentSessionFile,
    ];
    if (hasSession) args.push("-c");
    args.push(prompt);

    const textChunks: string[] = [];
    const startTime = Date.now();
    const timer = setInterval(() => {
      state.elapsed = Date.now() - startTime;
      updateWidget();
    }, 1000);

    return new Promise((resolve) => {
      const proc = spawn("pi", args, {
        stdio: ["ignore", "pipe", "pipe"],
        env: { ...process.env },
      });

      let buffer = "";
      proc.stdout!.setEncoding("utf-8");
      proc.stdout!.on("data", (chunk: string) => {
        buffer += chunk;
        const lines = buffer.split("\n");
        buffer = lines.pop() || "";
        for (const line of lines) {
          if (!line.trim()) continue;
          try {
            const event = JSON.parse(line);
            if (event.type === "message_update") {
              const delta = event.assistantMessageEvent;
              if (delta?.type === "text_delta" && delta.delta) {
                textChunks.push(delta.delta);
                const full = textChunks.join("");
                state.lastWork = full.split("\n").filter((l: string) => l.trim()).pop() || "";
                updateWidget();
              }
            }
          } catch {}
        }
      });
      proc.stderr!.setEncoding("utf-8");
      proc.stderr!.on("data", () => {});

      proc.on("close", (code) => {
        if (buffer.trim()) {
          try {
            const ev = JSON.parse(buffer);
            if (ev.type === "message_update" && ev.assistantMessageEvent?.type === "text_delta") {
              textChunks.push(ev.assistantMessageEvent.delta || "");
            }
          } catch {}
        }
        clearInterval(timer);
        state.elapsed = Date.now() - startTime;
        state.status = code === 0 ? "done" : "error";
        const output = textChunks.join("");
        state.lastWork = output.split("\n").filter((l: string) => l.trim()).pop() || "";
        updateWidget();
        resolve({ output, exitCode: code ?? 1, elapsed: state.elapsed });
      });

      proc.on("error", (err) => {
        clearInterval(timer);
        state.status = "error";
        state.lastWork = `Error: ${err.message}`;
        updateWidget();
        resolve({ output: `Error running ${step.name}: ${err.message}`, exitCode: 1, elapsed: Date.now() - startTime });
      });
    });
  }

  async function runPipeline(
    task: string,
    ctx: any,
  ): Promise<{ output: string; success: boolean; elapsed: number }> {
    const pipelineStart = Date.now();
    stepStates = resetStepStates();
    updateWidget();

    let input = task;
    const originalTask = task;

    for (let i = 0; i < STEPS.length; i++) {
      const step = STEPS[i];
      stepStates[i].status = "running";
      updateWidget();

      const resolvedPrompt = step.prompt
        .replace(/\$INPUT/g, input)
        .replace(/\$ORIGINAL/g, originalTask);

      const result = await runStep(step, i, resolvedPrompt, ctx);

      if (result.exitCode !== 0) {
        stepStates[i].status = "error";
        updateWidget();
        return {
          output: `Pipeline failed at step ${i + 1} (${step.name}): ${result.output}`,
          success: false,
          elapsed: Date.now() - pipelineStart,
        };
      }

      stepStates[i].status = "done";
      updateWidget();
      input = result.output;
    }

    return { output: input, success: true, elapsed: Date.now() - pipelineStart };
  }

  pi.registerTool({
    name: "run_pipeline",
    description: "Execute the Scout → Builder → Reviewer pipeline. Use for significant work: new features, refactors, multi-file changes. Work directly for simple questions or quick lookups.",
    parameters: Type.Object({
      task: Type.String({ description: "The task to pass through the pipeline" }),
    }),
    execute: async (_callId, args, _signal, _onUpdate, ctx) => {
      const { task } = args as { task: string };
      const result = await runPipeline(task, ctx);
      const truncated = result.output.length > 8000
        ? result.output.slice(0, 8000) + "\n\n... [truncated]"
        : result.output;
      const status = result.success ? "done" : "error";
      return {
        content: [{ type: "text", text: `Pipeline ${status} in ${Math.round(result.elapsed / 1000)}s.\n\n${truncated}` }],
      };
    },
  });

  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;
    sessionDir = path.join(ctx.cwd, ".pi", "agent-sessions");
    fs.mkdirSync(sessionDir, { recursive: true });
    try {
      for (const file of fs.readdirSync(sessionDir)) {
        if (file.startsWith("pipeline-") && file.endsWith(".json")) {
          fs.unlinkSync(path.join(sessionDir, file));
        }
      }
    } catch {}
    stepStates = resetStepStates();
    updateWidget();
    ctx.ui.setStatus("pipeline", "Pipeline: Scout → Builder → Reviewer");
    ctx.ui.notify(
      "Review Pipeline loaded.\n" +
      "Use run_pipeline for significant work, or work directly for simple tasks.\n" +
      "Pipeline: Scout → Builder → Reviewer",
      "info"
    );
  });

  pi.on("before_agent_start", async (event, _ctx) => {
    const stepDescriptions = STEPS.map((s, i) =>
      `${i + 1}. **${s.name.charAt(0).toUpperCase() + s.name.slice(1)}** — Tools: ${s.tools}`
    ).join("\n");

    return {
      systemPrompt: event.systemPrompt + `\n\n## Review Pipeline

You have access to run_pipeline which executes a three-step workflow:

${stepDescriptions}

Use run_pipeline for significant work. Work directly for simple questions.
$ORIGINAL (the user's task) is available to all steps for context.`,
    };
  });
}
```

---

## Testing It

```bash
pi -e extensions/review-pipeline.ts
```

**Test 1 — Pipeline for a real task:**
```
Add a comment to the top of extensions/minimal.ts explaining what it does.
```
The agent should recognize this as significant work and call `run_pipeline`. Watch the
three step cards progress from pending through running to done. The final reviewer output
tells you whether the comment is correct.

**Test 2 — Direct work for simple questions:**
```
What does the setFooter function do?
```
The agent should answer this directly without calling `run_pipeline`. The pipeline is
for implementation work, not questions.

**Test 3 — Watch variable substitution work:**
After a pipeline run, check the Pi conversation log. You should see the builder's prompt
contains scout's actual output (the `$INPUT` replacement), not the literal string "$INPUT".

**Test 4 — Fail-fast behavior:**
Temporarily break a step by setting its tools to an invalid value like `"fake-tool"`.
Run the pipeline. It should fail at that step, mark it as error, and stop without running
subsequent steps.

**Test 5 — Multi-file change:**
```
Create a new extension file at extensions/hello.ts that just prints "Hello" to the
status bar when the session starts.
```
This is a good multi-step task: scout identifies where minimal.ts is as a reference,
builder writes hello.ts following that pattern, reviewer checks it for correctness.

---

## Deep Dive: Pipeline vs. Dispatcher — When to Use Which

### The Three Orchestration Patterns

After four tutorials you have seen three distinct multi-agent patterns:

**1. Background Fire-and-Forget (Tutorial 8)**
```
User → /research → subprocess starts → main session stays live → result delivered later
```
Best for: Long-running independent research that should not block the main session.
Limitation: You cannot chain the result into another step automatically.

**2. Dispatcher (Tutorial 9)**
```
User → primary agent → dispatch_agent → specialist → result → primary agent → User
```
Best for: Dynamic delegation where the right specialist depends on the task.
Limitation: The LLM must reason about which agent to call; it may choose wrong sometimes.

**3. Pipeline (Tutorial 10)**
```
User → primary agent → run_pipeline → scout → builder → reviewer → result → primary agent → User
```
Best for: Repeatable multi-step workflows where the step order is always the same.
Limitation: Fixed sequence means it cannot adapt if the task does not need all three steps.

### Choosing the Right Pattern

Ask these questions:

- **Is the order fixed?** If yes, pipeline. If the order varies based on the task, dispatcher.
- **Should each step see the previous step's output?** If yes, pipeline. If steps are
  independent, parallel subagents (Tutorial 8 approach with Promise.all).
- **Should it block the main agent?** If the result is needed before continuing, use
  awaited dispatch. If it can run independently, fire-and-forget.
- **Is it a single long task or many parallel tasks?** Single sequential task → pipeline.
  Many parallel independent tasks → multiple background subagents.

### Combining Patterns

The patterns are not mutually exclusive. `agent-team.ts` and `agent-chain.ts` in this
project show combinations:

- `agent-team.ts` implements a dispatcher where the primary agent is restricted to
  `dispatch_agent` only, but the LLM dynamically chooses agents.
- `agent-chain.ts` implements a configurable pipeline loaded from YAML, where the chain
  definition can be switched at runtime.
- `pi-pi.ts` uses parallel fire-and-forget subagents for research (Tutorial 8 style),
  then aggregates all results before writing the final extension.

Real orchestration systems often combine all three: parallel research, sequential
pipeline, with a dispatcher deciding when to run which pipeline.

---

## Exercises

**Exercise 1 — Error recovery: retry once on failure.**
In `runPipeline`, if a step fails, retry it once before failing the whole pipeline:

```typescript
let result = await runStep(step, i, resolvedPrompt, ctx);
if (result.exitCode !== 0) {
  // Retry once
  state.lastWork = "Retrying...";
  updateWidget();
  result = await runStep(step, i, resolvedPrompt, ctx);
}
if (result.exitCode !== 0) {
  // Now fail
}
```

**Exercise 2 — Add a /pipeline command.**
Register a `/pipeline` command that shows the step history (elapsed times, success/fail,
last work line) for the most recent pipeline run. Use `ctx.ui.notify()` with a
formatted string.

**Exercise 3 — Add timing to widget cards.**
The `renderCard` function already shows elapsed time. Add a total elapsed time display
at the bottom of the widget for the full pipeline:

```typescript
// After rendering all cards, append a summary line
outputLines.push(
  theme.fg("dim", `Total: ${Math.round(totalElapsed / 1000)}s`)
);
```

**Exercise 4 — Make the pipeline configurable.**
Define the steps in a YAML file (`.pi/agents/pipeline.yaml`) instead of the TypeScript
constant. Use the YAML parser pattern from `agent-chain.ts`. Reload the steps in
`session_start`. This lets users define their own pipelines without editing TypeScript.

**Exercise 5 — Add a fourth step.**
After reviewer, add a "tester" step that writes a test for the code the builder
produced. The tester receives the reviewer's report as `$INPUT` and the original task
as `$ORIGINAL`. Verify that the step card appears in the widget and the `$INPUT`
substitution carries the reviewer's output correctly.

---

## Summary

You have built a complete 3-step agent pipeline and learned:

- Pipeline vs. dispatcher: sequential chaining vs. dynamic delegation
- `$INPUT` and `$ORIGINAL` variable substitution connects steps
- `for...of` with `await` is the only loop pattern that runs steps sequentially
- `resetStepStates()` on every pipeline run ensures the widget starts clean
- The primary agent keeps all tools; `run_pipeline` is an optional escalation path
- `before_agent_start` augmentation (not replacement) keeps Pi's default instructions
- `--append-system-prompt` injects specialist identity into each subprocess
- Session files with `fs.existsSync` check enable agents to continue across pipeline runs

The full production version with YAML configuration, chain switching, and detailed
progress cards lives in `extensions/agent-chain.ts`.

---

## Where to Go From Here

You have now completed all four advanced tutorials. Here is what to explore next:

**Read the production extensions:**
- `extensions/agent-team.ts` — full dispatcher with team YAML, grid dashboard, and
  context window tracking per agent
- `extensions/agent-chain.ts` — full pipeline with YAML chains, switchable at runtime
- `extensions/pi-pi.ts` — meta-agent that uses parallel subagents to research and then
  write new Pi extensions

**Combine what you have learned:**
Try building an extension that uses all three patterns: a dispatcher that can choose
between sending a task to a pipeline or to individual background subagents.

**Experiment with system prompts:**
The specialist prompts in your agents have an enormous effect on output quality. Experiment
with more detailed instructions, different phrasing, and role descriptions. Compare the
output quality against simpler prompts.

**Connect to real workflows:**
Add a Git integration step (a specialist that commits the builder's changes after the
reviewer approves). Add a "deploy" step that only runs if the reviewer reports no
Critical issues. Build a pipeline that integrates with your actual development workflow.

---

**Navigation:** [← Tutorial 9](./09-multi-agent-dispatcher.md)
