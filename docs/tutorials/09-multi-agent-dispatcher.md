# Tutorial 9: Building a Multi-Agent Dispatcher

**Prerequisites:** Tutorial 8 (background subagents). You must understand `spawn()`,
JSONL parsing, and the widget system before this tutorial.

**Navigation:** [← Tutorial 8](./08-background-subagent.md) | [Tutorial 10 →](./10-agent-pipeline.md)

---

## What You Will Build

A `mini-team` extension implementing the **dispatcher pattern**: the primary agent you
talk to has zero codebase tools. Its only tool is `dispatch_agent`. When you ask it to
do anything, it must choose a specialist and delegate.

Three specialists are defined:

| Specialist | Tools | Role |
|---|---|---|
| **scout** | `read, grep, find, ls` | Explores and reports; never modifies files |
| **builder** | `read, write, edit, bash, grep, find, ls` | Implements changes |
| **reviewer** | `read, bash, grep, find, ls` | Reviews code for correctness and style |

A status widget above the editor shows which agent is currently active:

```
Scout    ● running 5s   reading extensions/minimal.ts
Builder  ○ idle
Reviewer ○ idle
```

Session flow:
```
You: "Read the README and summarize it"
  → Dispatcher picks scout
  → scout reads README.md
  → scout's summary returned to dispatcher
  → Dispatcher presents summary to you
```

---

## Background: The Dispatcher Pattern

### Why No Codebase Tools on the Primary Agent?

This is the key insight of the dispatcher pattern. By calling:

```typescript
pi.setActiveTools(["dispatch_agent"]);
```

...the primary agent is **physically incapable** of reading, writing, or running
anything. It cannot cheat. It cannot shortcut. Its only mechanism for accomplishing
work is `dispatch_agent`.

This enforces a clean separation of concerns:

- **Primary agent** = coordinator. It understands the user's request, decomposes it
  into subtasks, and chooses specialists. It never touches the codebase directly.
- **Specialist agents** = workers. Each has a defined role and tool set. They do not
  coordinate or delegate — they just execute and report.

Without this constraint, a capable LLM will take the path of least resistance: use
whatever tools are directly available, skip the delegation step, and produce less
structured output. The restriction forces the architecture you designed.

### Dispatch Is Synchronous (From the Primary Agent's View)

Unlike Tutorial 8's fire-and-forget research, `dispatch_agent` in the dispatcher
pattern is **awaited**. The primary agent calls `dispatch_agent`, the extension spawns
a subprocess, waits for it to complete, and returns the specialist's output as the
tool result. The primary agent sees this as a single tool call with a result — it does
not know or care that another process was involved.

```
Primary agent turn:
  → calls dispatch_agent({ agent: "scout", task: "read README.md" })
  → subprocess spawns
  → subprocess reads README.md
  → subprocess finishes
  → tool result: "README says..."
  → primary agent processes result and responds to you
```

---

## What You Will Learn

- The dispatcher pattern: primary agent delegates, specialists execute
- `pi.setActiveTools(["dispatch_agent"])` to restrict the primary agent to delegation only
- Registering a tool (`pi.registerTool`) with a TypeBox parameter schema
- Awaited subprocess execution: `await` on a `Promise<>` that resolves when the process exits
- `--append-system-prompt` to inject specialist identity without replacing Pi's defaults
- `--session <file>` for agent memory across multiple dispatches
- `before_agent_start` for building a dynamic agent catalog in the system prompt
- A simple status widget showing all specialists

---

## Step 1: Create the File and Define Specialists

```typescript
/**
 * Mini Team — Dispatcher-only orchestrator with three specialists
 *
 * The primary agent has NO codebase tools. It MUST delegate everything
 * via the dispatch_agent tool.
 *
 * Usage: pi -e extensions/mini-team.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";
const { spawn } = require("child_process") as typeof import("child_process");
import * as fs from "fs";
import * as path from "path";
import * as os from "os";
```

`@sinclair/typebox` is the schema library used throughout Pi extensions to define tool
parameter schemas. `Type.Object()` and `Type.String()` create JSON Schema objects that
Pi uses to validate parameters and generate documentation for the LLM.

Define the specialists as a plain array of objects:

```typescript
interface AgentDef {
  name: string;
  description: string;
  tools: string;
  systemPrompt: string;
}

const AGENTS: AgentDef[] = [
  {
    name: "scout",
    description: "Fast reconnaissance — explores the codebase and reports findings",
    tools: "read,grep,find,ls",
    systemPrompt: `You are a scout agent. Your job is to explore the codebase and
report findings accurately and concisely. You do NOT modify files under any
circumstances. Read what is needed, then write a clear summary of what you found.
Focus on facts: file names, line counts, function signatures, key patterns.`,
  },
  {
    name: "builder",
    description: "Implementation — writes and modifies code thoroughly",
    tools: "read,write,edit,bash,grep,find,ls",
    systemPrompt: `You are a builder agent. Your job is to implement changes to
the codebase. Follow existing patterns. Write complete, correct code. Read
relevant files before modifying them. After making changes, summarize what
you changed and why. Do not over-explain — be direct.`,
  },
  {
    name: "reviewer",
    description: "Code review — checks for bugs, style issues, and correctness",
    tools: "read,bash,grep,find,ls",
    systemPrompt: `You are a code reviewer. Your job is to review code for correctness,
style issues, bugs, and maintainability problems. You do NOT modify files.
Report issues with: file name, line number, description of the problem, and
a suggested fix. Rate each issue as Critical, High, Medium, or Low.`,
  },
];
```

**Why hardcode agents instead of loading from files?** This tutorial prioritizes
simplicity. The production `agent-team.ts` loads agents from markdown files in
`.pi/agents/` directories, which makes agents user-configurable without touching
TypeScript. You can extend this tutorial in Exercise 4 to do the same.

---

## Step 2: Define Agent State

Each agent needs runtime state for the widget renderer:

```typescript
interface AgentState {
  def: AgentDef;
  status: "idle" | "running" | "done" | "error";
  task: string;
  elapsed: number;
  lastWork: string;          // last text line from the agent's output
  sessionFile: string | null; // path to the session file for conversation continuity
}
```

`sessionFile` is important: on the first dispatch, Pi creates a new session. On
subsequent dispatches to the same agent, we pass `--session <file>` and `-c` (continue)
so the agent remembers previous work within the same outer session.

---

## Step 3: Write the Extension Shell and Widget

```typescript
export default function (pi: ExtensionAPI) {
  const agentStates: Map<string, AgentState> = new Map();
  let widgetCtx: any = null;
  let sessionDir = "";

  // Initialize agent states from the AGENTS array
  function initAgents(cwd: string) {
    sessionDir = path.join(cwd, ".pi", "agent-sessions");
    fs.mkdirSync(sessionDir, { recursive: true });

    agentStates.clear();
    for (const def of AGENTS) {
      agentStates.set(def.name, {
        def,
        status: "idle",
        task: "",
        elapsed: 0,
        lastWork: "",
        sessionFile: null,
      });
    }
  }

  function updateWidget() {
    if (!widgetCtx) return;

    widgetCtx.ui.setWidget("mini-team", (_tui: any, theme: any) => {
      return {
        render(_width: number): string[] {
          const lines: string[] = [];

          for (const [name, state] of Array.from(agentStates.entries())) {
            const statusColor =
              state.status === "idle" ? "dim" :
              state.status === "running" ? "accent" :
              state.status === "done" ? "success" : "error";

            const statusIcon =
              state.status === "idle" ? "○" :
              state.status === "running" ? "●" :
              state.status === "done" ? "✓" : "✗";

            const elapsedStr = state.status !== "idle"
              ? ` ${Math.round(state.elapsed / 1000)}s`
              : "";

            // Capitalize: "scout" → "Scout"
            const displayName = name.charAt(0).toUpperCase() + name.slice(1);

            let line =
              theme.fg("accent", displayName.padEnd(10)) +
              theme.fg(statusColor, `${statusIcon} ${state.status}${elapsedStr}`);

            if (state.lastWork) {
              const preview = state.lastWork.length > 50
                ? state.lastWork.slice(0, 47) + "..."
                : state.lastWork;
              line += theme.fg("dim", `   ${preview}`);
            }

            lines.push(line);
          }

          return lines;
        },
        invalidate() {},
      };
    });
  }
```

The widget uses a `Map.entries()` loop to render one line per agent. The `padEnd(10)`
call right-pads the agent name with spaces to align the status columns. This is a
simple but effective layout technique for terminal UI.

---

## Step 4: Write the Dispatch Function

This is the core of the extension. Unlike Tutorial 8's fire-and-forget approach, this
function returns a `Promise` that resolves when the subprocess exits. The tool handler
`await`s this promise, so the LLM gets a synchronous-feeling tool call:

```typescript
  function dispatchTo(
    agentName: string,
    task: string,
    ctx: any,
  ): Promise<{ output: string; exitCode: number; elapsed: number }> {
    const state = agentStates.get(agentName);
    if (!state) {
      return Promise.resolve({
        output: `Unknown agent "${agentName}". Available: ${AGENTS.map(a => a.name).join(", ")}`,
        exitCode: 1,
        elapsed: 0,
      });
    }

    if (state.status === "running") {
      return Promise.resolve({
        output: `Agent "${agentName}" is already running. Wait for it to finish.`,
        exitCode: 1,
        elapsed: 0,
      });
    }

    // Update state to running
    state.status = "running";
    state.task = task;
    state.elapsed = 0;
    state.lastWork = "";
    updateWidget();

    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    // Session file path for this agent
    const agentSessionFile = path.join(sessionDir, `${agentName}.json`);

    const args = [
      "--mode", "json",
      "-p",
      "--no-extensions",
      "--model", model,
      "--tools", state.def.tools,
      "--thinking", "off",
      // Append the specialist's identity to Pi's default system prompt.
      // --append-system-prompt adds to the END, so Pi's tool descriptions remain intact.
      "--append-system-prompt", state.def.systemPrompt,
      // Use a persistent session file so this agent remembers previous dispatches
      "--session", agentSessionFile,
    ];

    // If the agent has run before in this outer session, continue the conversation
    if (state.sessionFile) {
      args.push("-c");
    }

    args.push(task);

    const textChunks: string[] = [];
    const startTime = Date.now();

    // Elapsed time timer — updates the widget every second
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
                // Update lastWork with the most recent non-empty output line
                const full = textChunks.join("");
                const last = full.split("\n").filter((l: string) => l.trim()).pop() || "";
                state.lastWork = last;
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
            const event = JSON.parse(buffer);
            if (event.type === "message_update") {
              const delta = event.assistantMessageEvent;
              if (delta?.type === "text_delta") textChunks.push(delta.delta || "");
            }
          } catch {}
        }

        clearInterval(timer);
        state.elapsed = Date.now() - startTime;
        state.status = code === 0 ? "done" : "error";

        // Mark session file as available for future -c (continue) invocations
        if (code === 0) {
          state.sessionFile = agentSessionFile;
        }

        const output = textChunks.join("");
        state.lastWork = output.split("\n").filter((l: string) => l.trim()).pop() || "";
        updateWidget();

        ctx.ui.notify(
          `${agentName} ${state.status} in ${Math.round(state.elapsed / 1000)}s`,
          state.status === "done" ? "success" : "error"
        );

        resolve({
          output,
          exitCode: code ?? 1,
          elapsed: state.elapsed,
        });
      });

      proc.on("error", (err) => {
        clearInterval(timer);
        state.status = "error";
        state.lastWork = `Error: ${err.message}`;
        updateWidget();
        resolve({
          output: `Error spawning agent: ${err.message}`,
          exitCode: 1,
          elapsed: Date.now() - startTime,
        });
      });
    });
  }
```

**The key difference from Tutorial 8:** `dispatchTo` returns a `Promise` that wraps the
entire subprocess lifecycle. The `resolve()` call inside `proc.on("close")` signals the
Promise completion. The tool handler `await`s this Promise, so the LLM waits for the
specialist to finish before it receives the tool result.

**`--append-system-prompt` vs. `--no-extensions` + custom prompt injection:**
`--append-system-prompt` adds text to the END of Pi's default system prompt. This
preserves Pi's tool descriptions and agent loop instructions. The specialist gets its
identity injected while still knowing how to use its tools. This is safer than
replacing the system prompt entirely.

---

## Step 5: Register the dispatch_agent Tool

Tools must be registered at the top level of the extension function (not inside an
event handler). Pi registers them once on startup:

```typescript
  pi.registerTool({
    name: "dispatch_agent",
    description: "Delegate a task to a specialist agent. Available agents: scout (read-only exploration), builder (write code and files), reviewer (code review only). The agent executes the task and returns its complete output.",
    parameters: Type.Object({
      agent: Type.String({
        description: "Agent name: 'scout', 'builder', or 'reviewer'",
      }),
      task: Type.String({
        description: "Complete task description for the agent to execute",
      }),
    }),
    execute: async (_callId, args, _signal, _onUpdate, ctx) => {
      const { agent, task } = args as { agent: string; task: string };

      const result = await dispatchTo(agent, task, ctx);

      const truncated = result.output.length > 8000
        ? result.output.slice(0, 8000) + "\n\n... [truncated]"
        : result.output;

      const status = result.exitCode === 0 ? "done" : "error";
      const elapsedSec = Math.round(result.elapsed / 1000);

      return {
        content: [{
          type: "text",
          text: `[${agent}] ${status} in ${elapsedSec}s\n\n${truncated}`,
        }],
      };
    },
  });
```

**Why is the `await` critical here?** The `execute` function is an `async` function that
the Pi framework awaits. If you fire the subprocess and return immediately, the LLM gets
an empty result before the specialist has done any work. The `await dispatchTo(...)` holds
the `execute` function open until the Promise resolves — until the specialist's process
exits and its output is collected.

---

## Step 6: Wire Up Session Start and Before Agent Start

```typescript
  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;
    initAgents(ctx.cwd);

    // This is the dispatcher pattern's enforcing line:
    // Strip all tools except dispatch_agent. The primary agent can ONLY delegate.
    pi.setActiveTools(["dispatch_agent"]);

    updateWidget();
    ctx.ui.setStatus("mini-team", "Team: scout, builder, reviewer");
    ctx.ui.notify(
      "Mini Team loaded.\nYou are a dispatcher — use dispatch_agent to delegate all work.\n" +
      "Agents: scout (read-only), builder (writes code), reviewer (read-only review)",
      "info"
    );
  });

  pi.on("before_agent_start", async (_event, _ctx) => {
    // Build a dynamic catalog describing each agent for the LLM
    const catalog = AGENTS.map((a) =>
      `### ${a.name}\n${a.description}\n**Tools:** ${a.tools}`
    ).join("\n\n");

    // Return a complete replacement system prompt for the dispatcher
    return {
      systemPrompt: `You are a dispatcher agent. You coordinate specialist agents.
You have NO direct access to the codebase. Your ONLY tool is dispatch_agent.

## How to Work
- Analyze the user's request
- Choose the right specialist agent(s)
- Dispatch tasks using dispatch_agent
- Review results and dispatch follow-up agents if needed
- Summarize the final outcome for the user

## Rules
- NEVER try to read, write, or run code directly — you have no such tools
- ALWAYS use dispatch_agent to get work done
- You can chain agents: scout first to explore, then builder to implement
- Keep tasks focused: one clear objective per dispatch

## Available Agents

${catalog}`,
    };
  });
} // end of export default function
```

**Why a full replacement here (not concatenation)?** The dispatcher agent is a
fundamentally different role from a normal Pi assistant. Pi's default system prompt
describes a general-purpose coder with access to many tools. That description is wrong
for the dispatcher, which has only one tool and a coordination role. Replacing the entire
system prompt ensures the LLM is not confused by conflicting identities.

This is one of the two cases where full replacement is correct. The other is in Tutorial 10.
For most persona-injection use cases (Tutorial 7), concatenation is safer.

---

## Full Final Code

```typescript
/**
 * Mini Team — Dispatcher-only orchestrator with three specialists
 *
 * Usage: pi -e extensions/mini-team.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";
const { spawn } = require("child_process") as typeof import("child_process");
import * as fs from "fs";
import * as path from "path";

interface AgentDef {
  name: string;
  description: string;
  tools: string;
  systemPrompt: string;
}

interface AgentState {
  def: AgentDef;
  status: "idle" | "running" | "done" | "error";
  task: string;
  elapsed: number;
  lastWork: string;
  sessionFile: string | null;
}

const AGENTS: AgentDef[] = [
  {
    name: "scout",
    description: "Fast reconnaissance — explores the codebase and reports findings",
    tools: "read,grep,find,ls",
    systemPrompt: `You are a scout agent. Explore the codebase and report findings
accurately and concisely. Do NOT modify files. Read what is needed, then write
a clear summary. Focus on facts: file names, line counts, function signatures,
key patterns.`,
  },
  {
    name: "builder",
    description: "Implementation — writes and modifies code thoroughly",
    tools: "read,write,edit,bash,grep,find,ls",
    systemPrompt: `You are a builder agent. Implement changes to the codebase.
Follow existing patterns. Write complete, correct code. Read relevant files
before modifying them. After making changes, summarize what you changed and why.`,
  },
  {
    name: "reviewer",
    description: "Code review — checks for bugs, style issues, and correctness",
    tools: "read,bash,grep,find,ls",
    systemPrompt: `You are a code reviewer. Review code for correctness, style
issues, bugs, and maintainability problems. Do NOT modify files. Report issues
with: file name, line number, description, and suggested fix. Rate each issue
as Critical, High, Medium, or Low.`,
  },
];

export default function (pi: ExtensionAPI) {
  const agentStates: Map<string, AgentState> = new Map();
  let widgetCtx: any = null;
  let sessionDir = "";

  function initAgents(cwd: string) {
    sessionDir = path.join(cwd, ".pi", "agent-sessions");
    fs.mkdirSync(sessionDir, { recursive: true });
    agentStates.clear();
    for (const def of AGENTS) {
      agentStates.set(def.name, {
        def, status: "idle", task: "", elapsed: 0, lastWork: "", sessionFile: null,
      });
    }
  }

  function updateWidget() {
    if (!widgetCtx) return;
    widgetCtx.ui.setWidget("mini-team", (_tui: any, theme: any) => {
      return {
        render(_width: number): string[] {
          const lines: string[] = [];
          for (const [name, state] of Array.from(agentStates.entries())) {
            const statusColor =
              state.status === "idle" ? "dim" :
              state.status === "running" ? "accent" :
              state.status === "done" ? "success" : "error";
            const statusIcon =
              state.status === "idle" ? "○" :
              state.status === "running" ? "●" :
              state.status === "done" ? "✓" : "✗";
            const elapsedStr = state.status !== "idle"
              ? ` ${Math.round(state.elapsed / 1000)}s` : "";
            const displayName = name.charAt(0).toUpperCase() + name.slice(1);
            let line =
              theme.fg("accent", displayName.padEnd(10)) +
              theme.fg(statusColor, `${statusIcon} ${state.status}${elapsedStr}`);
            if (state.lastWork) {
              const preview = state.lastWork.length > 50
                ? state.lastWork.slice(0, 47) + "..." : state.lastWork;
              line += theme.fg("dim", `   ${preview}`);
            }
            lines.push(line);
          }
          return lines;
        },
        invalidate() {},
      };
    });
  }

  function dispatchTo(
    agentName: string,
    task: string,
    ctx: any,
  ): Promise<{ output: string; exitCode: number; elapsed: number }> {
    const state = agentStates.get(agentName);
    if (!state) {
      return Promise.resolve({
        output: `Unknown agent "${agentName}". Available: ${AGENTS.map(a => a.name).join(", ")}`,
        exitCode: 1, elapsed: 0,
      });
    }
    if (state.status === "running") {
      return Promise.resolve({
        output: `Agent "${agentName}" is already running.`,
        exitCode: 1, elapsed: 0,
      });
    }

    state.status = "running";
    state.task = task;
    state.elapsed = 0;
    state.lastWork = "";
    updateWidget();

    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    const agentSessionFile = path.join(sessionDir, `${agentName}.json`);
    const args = [
      "--mode", "json", "-p", "--no-extensions",
      "--model", model,
      "--tools", state.def.tools,
      "--thinking", "off",
      "--append-system-prompt", state.def.systemPrompt,
      "--session", agentSessionFile,
    ];
    if (state.sessionFile) args.push("-c");
    args.push(task);

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
        if (code === 0) state.sessionFile = agentSessionFile;
        const output = textChunks.join("");
        state.lastWork = output.split("\n").filter((l: string) => l.trim()).pop() || "";
        updateWidget();
        ctx.ui.notify(
          `${agentName} ${state.status} in ${Math.round(state.elapsed / 1000)}s`,
          state.status === "done" ? "success" : "error"
        );
        resolve({ output, exitCode: code ?? 1, elapsed: state.elapsed });
      });

      proc.on("error", (err) => {
        clearInterval(timer);
        state.status = "error";
        state.lastWork = `Error: ${err.message}`;
        updateWidget();
        resolve({ output: `Error spawning agent: ${err.message}`, exitCode: 1, elapsed: Date.now() - startTime });
      });
    });
  }

  pi.registerTool({
    name: "dispatch_agent",
    description: "Delegate a task to a specialist agent. Available agents: scout (read-only exploration), builder (writes code), reviewer (code review). The agent executes and returns its complete output.",
    parameters: Type.Object({
      agent: Type.String({ description: "Agent name: 'scout', 'builder', or 'reviewer'" }),
      task: Type.String({ description: "Complete task description for the agent to execute" }),
    }),
    execute: async (_callId, args, _signal, _onUpdate, ctx) => {
      const { agent, task } = args as { agent: string; task: string };
      const result = await dispatchTo(agent, task, ctx);
      const truncated = result.output.length > 8000
        ? result.output.slice(0, 8000) + "\n\n... [truncated]"
        : result.output;
      const status = result.exitCode === 0 ? "done" : "error";
      return {
        content: [{ type: "text", text: `[${agent}] ${status} in ${Math.round(result.elapsed / 1000)}s\n\n${truncated}` }],
      };
    },
  });

  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;
    initAgents(ctx.cwd);
    pi.setActiveTools(["dispatch_agent"]);
    updateWidget();
    ctx.ui.setStatus("mini-team", "Team: scout, builder, reviewer");
    ctx.ui.notify(
      "Mini Team loaded.\nYou are a dispatcher — use dispatch_agent to delegate all work.\n" +
      "Agents: scout (read-only), builder (writes code), reviewer (read-only review)",
      "info"
    );
  });

  pi.on("before_agent_start", async (_event, _ctx) => {
    const catalog = AGENTS.map((a) =>
      `### ${a.name}\n${a.description}\n**Tools:** ${a.tools}`
    ).join("\n\n");

    return {
      systemPrompt: `You are a dispatcher agent. You coordinate specialist agents.
You have NO direct access to the codebase. Your ONLY tool is dispatch_agent.

## How to Work
- Analyze the user's request
- Choose the right specialist agent(s)
- Dispatch tasks using dispatch_agent
- Review results and dispatch follow-up agents if needed
- Summarize the final outcome for the user

## Rules
- NEVER try to read, write, or run code directly — you have no such tools
- ALWAYS use dispatch_agent to get work done
- You can chain agents: scout first to explore, then builder to implement
- Keep tasks focused: one clear objective per dispatch

## Available Agents

${catalog}`,
    };
  });
}
```

---

## Testing It

```bash
pi -e extensions/mini-team.ts
```

**Test 1 — Simple read task:**
```
Read the README.md and give me a one-paragraph summary.
```
The dispatcher must send this to scout. Watch the widget show scout transitioning from
idle to running and back. The dispatcher receives scout's summary and presents it.

**Test 2 — Verify the dispatcher cannot cheat:**
```
Just read extensions/minimal.ts yourself and tell me what it does.
```
The LLM has no `read` tool. It cannot comply directly. It should dispatch to scout
instead. If it tries to read the file directly, Pi will reject the tool call.

**Test 3 — Chained dispatch:**
```
Find a TypeScript file in extensions/ that has fewer than 50 lines, then add a comment
at the top explaining what it does.
```
The dispatcher should:
1. Dispatch to scout to find a short file.
2. Dispatch to builder to add the comment.
Watch the widget show scout completing, then builder starting.

**Test 4 — Review a file:**
```
Review extensions/minimal.ts for code quality and style issues.
```
The dispatcher should dispatch to reviewer. Reviewer should read the file and produce
a structured review.

**Test 5 — Confirm session memory:**
Ask scout to read a file. Then ask scout again to "look at the same file as before."
Because the session file is reused (via `-c`), the agent should remember the previous task.

---

## Deep Dive: Why Remove Tools From the Primary Agent?

### The Constraint Is the Feature

Calling `pi.setActiveTools(["dispatch_agent"])` might seem like an artificial restriction.
Why not give the dispatcher all tools and trust it to delegate appropriately?

The answer is that LLMs are capable and opportunistic. Given the option to read a file
directly or dispatch to a scout, the LLM will usually take the direct path — it is
simpler. This is fine for simple tasks, but it defeats the purpose of having specialists:

- **No role clarity.** If the dispatcher reads and writes directly, what exactly is the
  builder for?
- **No tool composition.** Specialists have carefully chosen tool sets. A dispatcher that
  reads files is now also acting as a scout but without the scout's focused system prompt.
- **Unpredictable behavior.** Sometimes it delegates, sometimes it works directly. The
  behavior depends on the LLM's mood, the phrasing of your request, and the complexity
  of the task.

By removing all codebase tools, you make delegation mandatory. The architecture is
enforced at the framework level, not just suggested in a system prompt. The LLM has
exactly one option: use `dispatch_agent`.

### The System Prompt Completes the Picture

Tool restriction alone is not enough. The LLM needs to understand *why* it only has
one tool. The `before_agent_start` handler provides this context:

```
You are a dispatcher agent. You coordinate specialist agents.
You have NO direct access to the codebase. Your ONLY tool is dispatch_agent.
```

Combined with the agent catalog showing what each specialist can do, the LLM understands
its role completely and reasons within that role rather than trying to work around it.

### Session Files Give Agents Memory

Without `--session <file>` and `-c`, every dispatch starts a fresh conversation. The
builder agent forgets what it built five minutes ago. The reviewer cannot refer back to
previous findings.

With session files:
- The session JSONL file stores the full conversation history.
- `-c` (continue) tells Pi to load this file and resume.
- The agent remembers everything from previous dispatches in the same outer session.

This is powerful for iterative workflows: scout → builder → reviewer, where the reviewer
can say "the code you wrote earlier has a bug on line 42."

On outer session restart (`session_start` fires again), the extension calls `initAgents()`
which resets all `sessionFile` references to `null`. This wipes agent memory — fresh
start for each outer session.

---

## Exercises

**Exercise 1 — Add a /team-status command.**
Register a `/team-status` command that prints the current status of all agents to the
notification area: name, status, run count, session state (new/resumed).

**Exercise 2 — Add a fourth specialist.**
Add a "documenter" agent with `read, write` tools and a system prompt focused on writing
documentation. Update the dispatcher's catalog and test it with a "write documentation
for X" request.

**Exercise 3 — Parallel dispatch.**
For tasks where the work is independent (e.g., "review all three files simultaneously"),
dispatch multiple agents concurrently using `Promise.all`:

```typescript
const [r1, r2] = await Promise.all([
  dispatchTo("reviewer", "review file A", ctx),
  dispatchTo("builder", "implement feature B", ctx),
]);
```

Be careful: the widget update function is shared. Make sure both agents have separate
state objects (they do — they are different entries in `agentStates`).

**Exercise 4 — Load agents from markdown files.**
Replace the hardcoded `AGENTS` array with a scanner that reads `.md` files from
`.pi/agents/` using the frontmatter format from `agent-team.ts`. This makes the team
configurable without editing TypeScript.

---

## Summary

You have learned:

- The dispatcher pattern: remove all codebase tools from the primary agent so it MUST delegate
- `pi.setActiveTools(["dispatch_agent"])` enforces the pattern at the framework level
- `pi.registerTool({ name, description, parameters, execute })` registers a tool the LLM can call
- `Type.Object({ field: Type.String({ description: "..." }) })` defines the parameter schema
- Awaited subprocess execution: `return new Promise((resolve) => { proc.on("close", () => resolve(...)) })`
- `--append-system-prompt` adds specialist identity to Pi's default system prompt
- `--session <file>` and `-c` give specialists persistent memory across dispatches
- Full system prompt replacement is appropriate when the agent has a fundamentally different role

The full production version with team selection, YAML team definitions, and a grid
dashboard lives in `extensions/agent-team.ts`.

---

**Navigation:** [← Tutorial 8](./08-background-subagent.md) | [Tutorial 10 →](./10-agent-pipeline.md)
