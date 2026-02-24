# Tutorial 8: Spawning Background Subagents

**Prerequisites:** Tutorial 7 (system prompt injection). You should understand event
handlers, command registration, and basic widget rendering.

**Navigation:** [← Tutorial 7](./07-system-prompt-injection.md) | [Tutorial 9 →](./09-multi-agent-dispatcher.md)

---

## What You Will Build

A `research` extension with a single `/research <topic>` command. When you type it:

1. A live progress widget appears above the editor showing elapsed time, tool call count,
   and a preview of what the subagent is currently doing.
2. A separate Pi process starts in the background and researches the topic.
3. The main session stays fully interactive — you can keep chatting while research runs.
4. When the subagent finishes, its findings are delivered as a follow-up message that
   the main agent can read and discuss.

Example session:

```
/research "explain how JavaScript closures work"
```

A widget appears:
```
────────────────────────────────────────────────────────
● Research  explain how JavaScript closures work   (8s)  | Tools: 3
  A closure is a function that retains access to its lexical scope...
────────────────────────────────────────────────────────
```

When done, the main agent receives:
```
Research finished "explain how JavaScript closures work" in 23s.

Result:
A closure in JavaScript is a function that captures variables from its
surrounding lexical scope at the time it was defined...
```

---

## Background: Why Subprocess Communication?

### The Problem With In-Process Agents

You might wonder: why spawn a separate process? Why not just call the AI API directly
from the extension? Two reasons:

**1. Pi already knows how to be an agent.** Pi handles the entire agentic loop —
system prompt construction, tool registration, tool execution, multi-turn conversation,
model selection, API retries. Re-implementing all of this inside an extension would
be enormous. By spawning another `pi` process, you get a full agent for free.

**2. Isolation.** If the subagent uses 100k tokens, crashes, or gets confused, it does
not affect the main session. Each `pi` process has its own context window, its own
tool instances, and its own event loop.

### The JSONL Event Stream

When Pi runs with `--mode json`, instead of rendering a terminal UI, it writes one JSON
object per line to stdout. Each line is a complete, parseable event. This is a standard
pattern called **JSONL** (JSON Lines) — one JSON value per line, newline-delimited.

Key event types:

```json
{"type":"message_update","assistantMessageEvent":{"type":"text_delta","delta":"Hello "}}
{"type":"message_update","assistantMessageEvent":{"type":"text_delta","delta":"world"}}
{"type":"tool_execution_start","toolName":"bash","toolCallId":"abc123"}
{"type":"tool_execution_end","toolName":"bash","toolCallId":"abc123"}
{"type":"agent_end"}
```

The parent process reads stdout line by line, buffers partial lines, and parses complete
JSON objects. This gives you a live stream of the subagent's activity.

### Fire-and-Forget vs. Await

There are two ways to use a subagent:

- **Fire-and-forget:** Start the process, return immediately, deliver results later via
  `pi.sendMessage()`. The main agent keeps running. Used by `/research` and `/sub`.
- **Await:** Start the process, block the current tool call until it finishes, return
  its output as the tool result. Used by `dispatch_agent` in `agent-team.ts`.

This tutorial uses fire-and-forget. Tutorial 9 covers the await pattern.

---

## What You Will Learn

- `child_process.spawn()` to launch Pi subprocesses
- Pi's `--mode json` flag for JSONL event streaming
- Parsing JSON events from stdout line by line
- Fire-and-forget process management
- `pi.sendMessage()` with `triggerTurn: true` to deliver results
- Live widget updates during subprocess execution
- Buffering partial stdout chunks

---

## Step 1: Create the File and Import Dependencies

```typescript
/**
 * Research — /research <topic> spawns a background Pi subagent
 *
 * The subagent researches the topic and delivers results when done.
 * A live widget shows progress while research runs.
 *
 * Usage: pi -e extensions/research.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
const { spawn } = require("child_process") as typeof import("child_process");
```

**Why `require` instead of `import`?** Pi extensions run through jiti (a just-in-time
TypeScript loader). Node.js built-in modules like `child_process` work most reliably
with `require()` in this environment. The cast `as typeof import("child_process")` gives
you TypeScript type checking — it tells the type checker to treat the result as if it
were a proper ES module import, so you get autocomplete and type safety.

---

## Step 2: Define the State Object

Before writing any handlers, define what you need to track about a running research job:

```typescript
interface ResearchState {
  status: "running" | "done" | "error";
  topic: string;
  textChunks: string[];   // accumulated text from message_update events
  toolCount: number;       // number of tool_execution_start events seen
  elapsed: number;         // milliseconds since the process started
}
```

This is the single source of truth for the widget renderer and the process event
handlers. Both read from and write to this object.

---

## Step 3: Write the Extension Shell

```typescript
export default function (pi: ExtensionAPI) {
  // One active research job at a time (for simplicity)
  let state: ResearchState | null = null;
  let widgetCtx: any = null;
```

`widgetCtx` stores the `ctx` object from the most recent event handler call. You need
`ctx` to call `ctx.ui.setWidget()`, but `ctx` is only available inside event handlers —
not at the top level of the extension. Storing it in a closure variable is the standard
pattern for accessing it from timer callbacks and process event handlers.

---

## Step 4: Write the Widget Renderer

```typescript
  function updateWidget() {
    if (!widgetCtx || !state) return;

    const key = "research";

    if (state.status !== "running" && state.status !== "done") {
      // Remove the widget on error or if state is cleared
      widgetCtx.ui.setWidget(key, undefined);
      return;
    }

    widgetCtx.ui.setWidget(key, (_tui: any, theme: any) => {
      return {
        render(width: number): string[] {
          const lines: string[] = [];

          // Status icon and color
          const statusColor = state!.status === "running" ? "accent" : "success";
          const statusIcon = state!.status === "running" ? "●" : "✓";

          // Topic preview (truncate if too long)
          const topicPreview = state!.topic.length > 40
            ? state!.topic.slice(0, 37) + "..."
            : state!.topic;

          // Header line: icon, topic, elapsed time, tool count
          const elapsedSec = Math.round(state!.elapsed / 1000);
          lines.push(
            theme.fg(statusColor, `${statusIcon} Research`) +
            theme.fg("dim", `  ${topicPreview}`) +
            theme.fg("dim", `  (${elapsedSec}s)`) +
            theme.fg("dim", `  | Tools: ${state!.toolCount}`)
          );

          // Preview of the last line of output
          const fullText = state!.textChunks.join("");
          const lastLine = fullText
            .split("\n")
            .filter((l) => l.trim())
            .pop() || "";

          if (lastLine) {
            const trimmed = lastLine.length > width - 4
              ? lastLine.slice(0, width - 7) + "..."
              : lastLine;
            lines.push(theme.fg("muted", `  ${trimmed}`));
          }

          return lines;
        },
        invalidate() {}, // required by the widget interface; no-op is fine
      };
    });
  }
```

The widget renderer is a function that returns an object with a `render(width)` method.
Pi calls `render` every time it needs to redraw that area of the screen. The `width`
parameter tells you how many terminal columns are available. Always respect width
constraints — overshooting causes visual glitches.

`theme.fg("accent", text)` wraps `text` in ANSI escape codes for the current theme's
accent color. Using theme colors (rather than hardcoded ANSI codes) makes your widget
look correct in all Pi themes.

---

## Step 5: Parse JSONL Events From the Subprocess

```typescript
  function processLine(line: string) {
    if (!line.trim() || !state) return;

    try {
      const event = JSON.parse(line);

      if (event.type === "message_update") {
        // message_update carries a streaming text delta from the LLM
        const delta = event.assistantMessageEvent;
        if (delta?.type === "text_delta" && delta.delta) {
          state.textChunks.push(delta.delta);
          updateWidget();
        }
      } else if (event.type === "tool_execution_start") {
        // A tool call began — increment the count and refresh the widget
        state.toolCount++;
        updateWidget();
      }
      // We ignore other event types (tool_execution_end, message_end, etc.)
      // for this simplified example
    } catch {
      // Ignore malformed JSON lines — this happens with stderr output
      // or partial lines that arrive at chunk boundaries
    }
  }
```

**Why the try/catch?** stdout is a byte stream. You split it by `\n`, but sometimes a
JSON object spans two chunks. Partial lines are not valid JSON. The `try/catch` silently
discards them — you will see the rest of the object in the next chunk and process it
then. Alternatively, you buffer until `\n` before parsing; see Step 6.

---

## Step 6: Spawn the Subprocess

```typescript
  function spawnResearch(topic: string, ctx: any) {
    widgetCtx = ctx;

    state = {
      status: "running",
      topic,
      textChunks: [],
      toolCount: 0,
      elapsed: 0,
    };

    updateWidget();

    // Determine which model to use — inherit from the main session
    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    // Build the argument list for the pi subprocess
    const args = [
      "--mode", "json",         // output JSONL events instead of rendering a TUI
      "-p",                     // print mode: non-interactive, single prompt
      "--no-extensions",        // do not load any extensions in the subprocess
      "--no-session",           // do not save/restore conversation history
      "--model", model,         // use the same model as the main session
      "--tools", "read,bash,grep,find,ls",  // research-appropriate tools
      "--thinking", "off",      // disable extended thinking for speed
      topic,                    // the actual prompt
    ];

    const proc = spawn("pi", args, {
      stdio: ["ignore", "pipe", "pipe"], // ignore stdin, capture stdout and stderr
      env: { ...process.env },           // inherit environment (API keys, PATH, etc.)
    });

    // ── Elapsed time timer ────────────────────────────────────────────────────

    const startTime = Date.now();
    const timer = setInterval(() => {
      if (state) {
        state.elapsed = Date.now() - startTime;
        updateWidget();
      }
    }, 1000);

    // ── stdout: JSONL event stream ────────────────────────────────────────────

    // Buffer handles the case where a JSON object spans two data chunks.
    // We accumulate data until we see a newline, then parse the complete line.
    let buffer = "";

    proc.stdout!.setEncoding("utf-8");
    proc.stdout!.on("data", (chunk: string) => {
      buffer += chunk;
      const lines = buffer.split("\n");
      // The last element may be a partial line — keep it in the buffer
      buffer = lines.pop() || "";
      for (const line of lines) {
        processLine(line);
      }
    });

    // ── stderr: log but ignore ────────────────────────────────────────────────

    proc.stderr!.setEncoding("utf-8");
    proc.stderr!.on("data", (_chunk: string) => {
      // Pi writes informational messages to stderr (model name, session info).
      // We discard them here. In a production extension you might log them.
    });

    // ── Process exit ──────────────────────────────────────────────────────────

    proc.on("close", (code) => {
      // Process any remaining buffered data
      if (buffer.trim()) processLine(buffer);

      clearInterval(timer);

      if (!state) return;

      state.elapsed = Date.now() - startTime;
      state.status = code === 0 ? "done" : "error";

      // Brief widget flash showing completion, then remove it after a moment
      updateWidget();

      // Collect the full research output
      const result = state.textChunks.join("");
      const elapsedSec = Math.round(state.elapsed / 1000);

      // Notify the user in the status bar
      ctx.ui.notify(
        `Research finished in ${elapsedSec}s`,
        state.status === "done" ? "success" : "error"
      );

      // Deliver the result to the main agent as a follow-up message.
      // triggerTurn: true causes the main agent to process the message immediately.
      pi.sendMessage(
        {
          customType: "research-result",
          content: `Research on "${topic}" completed in ${elapsedSec}s.\n\nFindings:\n${result.slice(0, 8000)}${result.length > 8000 ? "\n\n... [truncated]" : ""}`,
          display: true,
        },
        { deliverAs: "followUp", triggerTurn: true }
      );

      // Remove the widget after delivery
      setTimeout(() => {
        widgetCtx?.ui.setWidget("research", undefined);
      }, 2000);
    });

    proc.on("error", (err) => {
      clearInterval(timer);
      if (state) {
        state.status = "error";
        updateWidget();
      }
      ctx.ui.notify(`Research failed: ${err.message}`, "error");
    });
  }
```

**Key details to understand:**

`stdio: ["ignore", "pipe", "pipe"]` — The three elements correspond to stdin, stdout,
and stderr. `"ignore"` means no stdin (the subprocess cannot receive keyboard input).
`"pipe"` means we capture the stream as a Node.js Readable. This is how `proc.stdout`
and `proc.stderr` become readable.

`{ ...process.env }` — Spreads all current environment variables into the child
process. This is essential: without it, the subprocess would not have `ANTHROPIC_API_KEY`,
`PATH`, or any other environment variable. The subprocess is a clean process — it
inherits nothing unless you explicitly pass it.

`--no-extensions` — Prevents the subprocess from loading your extension file again,
which would cause the subprocess to register another `/research` command, which would
try to spawn another subprocess, and so on. Always pass `--no-extensions` to subagents
unless you specifically need them.

`--no-session` — Prevents the subprocess from loading or saving conversation history.
The subagent starts fresh every time.

---

## Step 7: Register the /research Command

```typescript
  pi.registerCommand("research", {
    description: "Research a topic in the background: /research <topic>",
    handler: async (args, ctx) => {
      const topic = args?.trim();

      if (!topic) {
        ctx.ui.notify("Usage: /research <topic>", "error");
        return;
      }

      if (state?.status === "running") {
        ctx.ui.notify("Research already running. Wait for it to finish.", "warning");
        return;
      }

      ctx.ui.notify(`Starting research on: ${topic}`, "info");

      // Fire-and-forget: spawnResearch returns immediately.
      // The process runs in the background; this handler returns before it finishes.
      spawnResearch(topic, ctx);
    },
  });

  // Keep widgetCtx up to date when new sessions start
  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;
    state = null;
  });
} // end of export default function
```

The handler calls `spawnResearch(topic, ctx)` without `await`. This is intentional —
the handler returns immediately, and the subprocess runs asynchronously. The command
handler completing does not mean the research is done; it means the process has been
started.

---

## Full Final Code

```typescript
/**
 * Research — /research <topic> spawns a background Pi subagent
 *
 * Usage: pi -e extensions/research.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
const { spawn } = require("child_process") as typeof import("child_process");

interface ResearchState {
  status: "running" | "done" | "error";
  topic: string;
  textChunks: string[];
  toolCount: number;
  elapsed: number;
}

export default function (pi: ExtensionAPI) {
  let state: ResearchState | null = null;
  let widgetCtx: any = null;

  function updateWidget() {
    if (!widgetCtx || !state) return;
    const key = "research";
    if (state.status !== "running" && state.status !== "done") {
      widgetCtx.ui.setWidget(key, undefined);
      return;
    }
    widgetCtx.ui.setWidget(key, (_tui: any, theme: any) => {
      return {
        render(width: number): string[] {
          const lines: string[] = [];
          const statusColor = state!.status === "running" ? "accent" : "success";
          const statusIcon = state!.status === "running" ? "●" : "✓";
          const topicPreview = state!.topic.length > 40
            ? state!.topic.slice(0, 37) + "..."
            : state!.topic;
          const elapsedSec = Math.round(state!.elapsed / 1000);
          lines.push(
            theme.fg(statusColor, `${statusIcon} Research`) +
            theme.fg("dim", `  ${topicPreview}`) +
            theme.fg("dim", `  (${elapsedSec}s)`) +
            theme.fg("dim", `  | Tools: ${state!.toolCount}`)
          );
          const fullText = state!.textChunks.join("");
          const lastLine = fullText.split("\n").filter((l) => l.trim()).pop() || "";
          if (lastLine) {
            const trimmed = lastLine.length > width - 4
              ? lastLine.slice(0, width - 7) + "..."
              : lastLine;
            lines.push(theme.fg("muted", `  ${trimmed}`));
          }
          return lines;
        },
        invalidate() {},
      };
    });
  }

  function processLine(line: string) {
    if (!line.trim() || !state) return;
    try {
      const event = JSON.parse(line);
      if (event.type === "message_update") {
        const delta = event.assistantMessageEvent;
        if (delta?.type === "text_delta" && delta.delta) {
          state.textChunks.push(delta.delta);
          updateWidget();
        }
      } else if (event.type === "tool_execution_start") {
        state.toolCount++;
        updateWidget();
      }
    } catch {}
  }

  function spawnResearch(topic: string, ctx: any) {
    widgetCtx = ctx;
    state = { status: "running", topic, textChunks: [], toolCount: 0, elapsed: 0 };
    updateWidget();

    const model = ctx.model
      ? `${ctx.model.provider}/${ctx.model.id}`
      : "openrouter/google/gemini-2-flash";

    const args = [
      "--mode", "json", "-p", "--no-extensions", "--no-session",
      "--model", model,
      "--tools", "read,bash,grep,find,ls",
      "--thinking", "off",
      topic,
    ];

    const proc = spawn("pi", args, {
      stdio: ["ignore", "pipe", "pipe"],
      env: { ...process.env },
    });

    const startTime = Date.now();
    const timer = setInterval(() => {
      if (state) { state.elapsed = Date.now() - startTime; updateWidget(); }
    }, 1000);

    let buffer = "";
    proc.stdout!.setEncoding("utf-8");
    proc.stdout!.on("data", (chunk: string) => {
      buffer += chunk;
      const lines = buffer.split("\n");
      buffer = lines.pop() || "";
      for (const line of lines) processLine(line);
    });

    proc.stderr!.setEncoding("utf-8");
    proc.stderr!.on("data", () => {});

    proc.on("close", (code) => {
      if (buffer.trim()) processLine(buffer);
      clearInterval(timer);
      if (!state) return;
      state.elapsed = Date.now() - startTime;
      state.status = code === 0 ? "done" : "error";
      updateWidget();

      const result = state.textChunks.join("");
      const elapsedSec = Math.round(state.elapsed / 1000);

      ctx.ui.notify(
        `Research finished in ${elapsedSec}s`,
        state.status === "done" ? "success" : "error"
      );

      pi.sendMessage(
        {
          customType: "research-result",
          content: `Research on "${topic}" completed in ${elapsedSec}s.\n\nFindings:\n${result.slice(0, 8000)}${result.length > 8000 ? "\n\n... [truncated]" : ""}`,
          display: true,
        },
        { deliverAs: "followUp", triggerTurn: true }
      );

      setTimeout(() => {
        widgetCtx?.ui.setWidget("research", undefined);
      }, 2000);
    });

    proc.on("error", (err) => {
      clearInterval(timer);
      if (state) { state.status = "error"; updateWidget(); }
      ctx.ui.notify(`Research failed: ${err.message}`, "error");
    });
  }

  pi.registerCommand("research", {
    description: "Research a topic in the background: /research <topic>",
    handler: async (args, ctx) => {
      const topic = args?.trim();
      if (!topic) { ctx.ui.notify("Usage: /research <topic>", "error"); return; }
      if (state?.status === "running") {
        ctx.ui.notify("Research already running. Wait for it to finish.", "warning");
        return;
      }
      ctx.ui.notify(`Starting research on: ${topic}`, "info");
      spawnResearch(topic, ctx);
    },
  });

  pi.on("session_start", async (_event, ctx) => {
    widgetCtx = ctx;
    state = null;
  });
}
```

---

## Testing It

```bash
pi -e extensions/research.ts
```

**Test 1 — Basic research:**
```
/research "explain how JavaScript closures work"
```
Watch the widget appear. The status icon pulses. The tool count increments as the
subagent reads files or runs bash. After 20–60 seconds, the result arrives as a message.

**Test 2 — Research while chatting:**
```
/research "explain async/await in TypeScript"
```
While research is running, type: `What is 2 + 2?`
The main agent responds immediately. Research continues in the background.

**Test 3 — Codebase research:**
```
/research "summarize all the extensions in this project and what each one does"
```
The subagent has `read`, `bash`, `grep`, `find`, `ls` available. Watch it scan files.

**Test 4 — Confirm trigger works:**
After research finishes, the main agent should automatically respond to the research
result (because `triggerTurn: true` causes it to process the message). Watch for the
main agent's acknowledgement.

---

## Deep Dive: The JSONL Event Stream

### Complete Event Reference

When Pi runs with `--mode json`, every internal event is emitted to stdout as a JSON line.
Here are the key ones for subagent integration:

```
message_update
  Fires repeatedly as the LLM generates tokens.
  Structure: { type: "message_update", assistantMessageEvent: { type: "text_delta", delta: "..." } }
  Also fires with type: "thinking_delta" during thinking mode.

tool_execution_start
  Fires when the LLM calls a tool and execution begins.
  Structure: { type: "tool_execution_start", toolName: "bash", toolCallId: "..." }

tool_execution_end
  Fires when a tool call completes.
  Structure: { type: "tool_execution_end", toolName: "bash", toolCallId: "...", result: {...} }

message_end
  Fires when the LLM finishes one response (before any tool results are sent back).
  Structure: { type: "message_end", message: { usage: { input: N, output: N } } }

agent_end
  Fires when the entire agent loop completes — no more turns will happen.
  Structure: { type: "agent_end", messages: [...] }
```

### Why Line Buffering Matters

`proc.stdout.on("data")` fires whenever data arrives from the process, which may be
any number of bytes. A single `data` event might deliver:
- An incomplete JSON object: `{"type":"message_upd`
- Multiple complete objects and a partial: `{"type":"agent_end"}\n{"type":"mes`
- Many complete lines at once

The buffering pattern handles all of these correctly:

```typescript
let buffer = "";
proc.stdout.on("data", (chunk) => {
  buffer += chunk;               // accumulate
  const lines = buffer.split("\n");
  buffer = lines.pop() || "";   // last element is incomplete; keep buffering
  for (const line of lines) {   // process all complete lines
    processLine(line);
  }
});
// On close, process whatever is left in the buffer
if (buffer.trim()) processLine(buffer);
```

The `lines.pop()` call removes the last element from the array and returns it. After
splitting `"line1\nline2\npartial"` by `"\n"`, you get `["line1", "line2", "partial"]`.
`pop()` removes and returns `"partial"`, leaving `["line1", "line2"]` to process.
`"partial"` goes back into `buffer` to await the rest.

### sendMessage Options

```typescript
pi.sendMessage(
  {
    customType: "research-result",  // identifies the message type for filtering
    content: "the text",            // what the agent receives
    display: true,                  // show this message in the conversation UI
  },
  {
    deliverAs: "followUp",          // add to conversation as a follow-up, not inline
    triggerTurn: true,              // immediately trigger the agent to process it
  }
);
```

`triggerTurn: true` is the key option. Without it, the message is added to the queue
but the agent does not act on it until the user sends another message. With it, the
agent wakes up and responds immediately — as if you had typed the research result
yourself.

---

## Exercises

**Exercise 1 — Cancel running research.**
Add a `/research-cancel` command that kills the running subprocess:

```typescript
let proc: any = null; // store the process reference

// In spawnResearch, after calling spawn():
proc = spawn("pi", args, ...);

// New command:
pi.registerCommand("research-cancel", {
  description: "Cancel running research",
  handler: async (_args, ctx) => {
    if (!proc || state?.status !== "running") {
      ctx.ui.notify("No research is currently running.", "info");
      return;
    }
    proc.kill("SIGTERM");
    ctx.ui.notify("Research cancelled.", "warning");
  },
});
```

**Exercise 2 — Multiple concurrent research tasks.**
Use a `Map<number, ResearchState>` (keyed by an auto-incrementing ID) instead of a
single `state` variable. Give each task its own widget key (`research-1`, `research-2`,
etc.) and update them independently.

**Exercise 3 — Save research to a file.**
When research completes, write the full result to `.pi/research/<timestamp>-<topic>.md`
using `fs.writeFileSync()`. Then notify the user of the file path so they can open it
later or feed it to another agent.

**Exercise 4 — Progress percentage.**
Parse `message_end` events, which include token usage. Calculate how many tokens the
subagent has consumed and show a rough progress bar in the widget. (Note: you need to
know the model's context window size, which you can get from `ctx.model.contextWindow`.)

---

## Summary

You have learned:

- Spawn a Pi subprocess with `spawn("pi", ["--mode", "json", "-p", "--no-extensions", ...], ...)`
- Pi's `--mode json` flag turns the terminal UI off and emits JSONL events to stdout
- Buffer stdout chunks and split on `\n` to get complete JSON lines
- Parse `message_update` events for text deltas and `tool_execution_start` for tool counts
- Fire-and-forget: call `spawnResearch()` without `await`, let the process run independently
- Deliver results with `pi.sendMessage({ content: ... }, { deliverAs: "followUp", triggerTurn: true })`
- Always pass `{ ...process.env }` to the subprocess so it inherits your API keys

The full production version with persistent sessions, multiple concurrent subagents, and
continue support lives in `extensions/subagent-widget.ts`.

---

**Navigation:** [← Tutorial 7](./07-system-prompt-injection.md) | [Tutorial 9 →](./09-multi-agent-dispatcher.md)
