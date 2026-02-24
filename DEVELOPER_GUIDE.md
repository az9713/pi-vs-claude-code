# Pi Extension Developer Guide

A comprehensive reference for developers who want to maintain and extend this project. Assumes familiarity with C/C++ or Java but not with TypeScript, modern JavaScript tooling, or AI agent development.

---

## Table of Contents

1. [Developer Prerequisites](#1-developer-prerequisites)
2. [Project Setup for Development](#2-project-setup-for-development)
3. [How Pi Extensions Work — The Mental Model](#3-how-pi-extensions-work--the-mental-model)
4. [Extension Anatomy — Line by Line](#4-extension-anatomy--line-by-line)
5. [The Extension API Reference](#5-the-extension-api-reference)
6. [How to Create a New Extension](#6-how-to-create-a-new-extension)
7. [How to Create a Multi-Agent Extension](#7-how-to-create-a-multi-agent-extension)
8. [How to Add Safety Rules](#8-how-to-add-safety-rules)
9. [How Themes Work](#9-how-themes-work)
10. [Testing and Debugging](#10-testing-and-debugging)
11. [Code Style and Conventions](#11-code-style-and-conventions)
12. [Common Patterns](#12-common-patterns)
13. [Glossary](#13-glossary)

---

## 1. Developer Prerequisites

### What You Need to Know

#### TypeScript for C/Java Developers

TypeScript is JavaScript with a type system bolted on top. If you know Java, think of it this way:

- JavaScript is like Java without the compiler — it runs in a runtime (Node.js or Bun) and is dynamically typed.
- TypeScript adds a static type checker on top. It reads `.ts` files, checks types at "compile time," and outputs plain `.js`.
- In this project, there is no explicit compile step. Pi uses **jiti**, a just-in-time TypeScript runner, which compiles on the fly when it loads your `.ts` file.

Key differences from Java:

```typescript
// Java: String name = "Alice";
// TypeScript:
let name: string = "Alice";  // explicit type annotation
let name2 = "Alice";         // type inferred — TypeScript figures out it's a string

// Java: void doThing(String s) { ... }
// TypeScript:
function doThing(s: string): void { ... }

// Java: interface Foo { ... }
// TypeScript has both interface and type alias:
interface Foo { name: string; age: number; }
type Bar = { name: string; age: number; };  // equivalent in most cases

// Java: List<String> = new ArrayList<>();
// TypeScript:
const items: string[] = [];
const items2: Array<string> = [];  // same thing

// Union types (no Java equivalent):
type Status = "idle" | "running" | "done";  // only these three string values allowed

// Optional fields — the ? means the field may be absent:
interface Config { name: string; timeout?: number; }

// Null/undefined — TypeScript distinguishes between them, Java only has null
let x: string | undefined = undefined;
let y: string | null = null;
```

The `type` and `import type` keywords: when you see `import type { Foo }`, it means you are only importing the type for compile-time checking. The import disappears at runtime. This is common for interface definitions.

#### ES Modules (import/export)

This project uses ES modules, the modern JavaScript module system.

```typescript
// C: #include "foo.h"
// Java: import com.example.Foo;
// TypeScript ES module:
import { specificThing } from "./other-file.ts";
import defaultExport from "./other-file.ts";
import type { TypeOnly } from "./other-file.ts";  // compile-time only

// Exporting — what other files can use:
export function myFunction() { ... }
export const MY_CONSTANT = 42;
export default function mainEntry() { ... }  // one default per file
```

The `export default function` pattern is critical here — every extension exports exactly one default function. That is the entry point Pi calls.

#### Async/Await

JavaScript is single-threaded but non-blocking. Instead of threads blocking on I/O, you use asynchronous operations.

Java developers: think of `async/await` as syntactic sugar over `CompletableFuture`, but far more readable.

```typescript
// Marking a function as async — it returns a Promise automatically
async function fetchData(): Promise<string> {
    // await suspends this function until the Promise resolves
    // The runtime runs other code while waiting — not a blocked thread
    const result = await someAsyncOperation();
    return result;  // TypeScript wraps this in Promise<string> automatically
}

// Error handling — same try/catch you know:
async function riskyOp(): Promise<void> {
    try {
        await mightFail();
    } catch (err) {
        console.error("Failed:", err);
    }
}

// Running multiple async operations in parallel:
const [a, b] = await Promise.all([opA(), opB()]);

// Running in parallel but not failing if one errors:
const results = await Promise.allSettled([opA(), opB()]);
```

The `void` keyword before a function call (e.g., `void asyncFn()`) is a deliberate fire-and-forget: start the async operation, do not wait for it, and explicitly discard the returned Promise. You will see this in `purpose-gate.ts`.

#### What You Need to Install

1. **Bun** — the package manager and runtime for this project. Install from https://bun.sh.
   - Do not use npm, yarn, or pnpm. Only `bun`.

2. **just** — a command runner (like `make`, but saner). Install from https://just.systems.
   - On macOS: `brew install just`
   - On Linux: `cargo install just` or distro package manager

3. **Pi Coding Agent** — the AI agent this project extends. Obtain and install per the Pi documentation.
   - `pi` must be on your PATH for extensions to run and for multi-agent spawning to work.

---

## 2. Project Setup for Development

### Clone and Install

```bash
git clone <repo-url>
cd pi-vs-claude-code-main
bun install
```

`bun install` reads `package.json` and installs the single dependency (`yaml` library). It is fast.

### File Structure

```
pi-vs-claude-code-main/
├── extensions/          # All Pi extension source files (.ts)
│   ├── minimal.ts           # Simplest extension — good starting point
│   ├── themeMap.ts          # Shared helper — not an extension itself
│   ├── tilldone.ts          # Task management extension
│   ├── theme-cycler.ts      # Theme switching extension
│   ├── damage-control.ts    # Safety rules extension
│   ├── agent-team.ts        # Multi-agent dispatcher
│   ├── agent-chain.ts       # Sequential pipeline orchestrator
│   ├── subagent-widget.ts   # Background subagent spawner
│   ├── pi-pi.ts             # Meta-agent for building Pi agents
│   ├── purpose-gate.ts      # Forces purpose declaration before work
│   ├── tool-counter.ts      # Footer with token/cost/tool stats
│   ├── tool-counter-widget.ts  # Widget showing per-tool call counts
│   ├── session-replay.ts    # Scrollable session history overlay
│   ├── system-select.ts     # Pick agent persona as system prompt
│   ├── cross-agent.ts       # Load commands from .claude/.gemini/ dirs
│   └── pure-focus.ts        # Stripped-down minimal UI
│
├── .pi/
│   ├── themes/          # Theme JSON files (11 themes)
│   │   ├── synthwave.json
│   │   ├── dracula.json
│   │   └── ...
│   └── agents/          # Agent definition .md files
│       ├── scout.md
│       ├── builder.md
│       ├── planner.md
│       ├── teams.yaml       # Team groupings for agent-team extension
│       └── pi-pi/           # Experts for the pi-pi meta-agent
│           ├── ext-expert.md
│           └── ...
│
├── specs/               # Feature specification documents
├── justfile             # Task definitions (run with `just <name>`)
├── package.json         # Dependencies
└── CLAUDE.md            # Short project conventions
```

### How to Run Things

The `justfile` contains named recipes. Run `just` with no arguments to list them all.

```bash
# List available recipes
just

# Run the minimal extension
just ext-minimal

# Run the tilldone extension
just ext-tilldone

# Run any extension directly (bypassing just)
pi -e extensions/minimal.ts

# Stack multiple extensions
pi -e extensions/tilldone.ts -e extensions/theme-cycler.ts

# Open every extension in separate terminal windows (macOS only)
just all
```

Extensions are loaded in the order they are specified on the command line. When stacked, each extension's `session_start` handler fires (first to last). The theme system uses this order to ensure the primary extension sets the theme.

---

## 3. How Pi Extensions Work — The Mental Model

### The Big Picture

Pi is an interactive AI coding agent. Extensions let you customize its behavior: add tools the AI can call, register slash commands for the user, intercept events, and draw custom UI elements.

Pi loads your `.ts` file at runtime using **jiti**, a just-in-time TypeScript compiler. There is no build step. You edit the file, restart Pi, and your changes are live.

### The Entry Point

Every extension is a single `.ts` file that exports one default function:

```typescript
export default function (pi: ExtensionAPI) {
    // Your entire extension lives here
}
```

Pi calls this function once at startup, passing you the `ExtensionAPI` object. You use that object to register everything: tools, commands, shortcuts, and event handlers.

### The Callback Model

For C developers: this is exactly like registering callback functions with a framework. You hand Pi a function pointer, and Pi calls it when the relevant event occurs.

For Java developers: this is like implementing listener interfaces and registering them with an event bus. `pi.on("session_start", handler)` is analogous to `eventBus.subscribe(SessionStartEvent.class, handler)`.

```
Your extension file
        |
        v
Pi calls your default function with (pi: ExtensionAPI)
        |
        v
You register handlers: pi.on("session_start", ...), pi.registerTool(...), etc.
        |
        v
Pi runs the session. When events occur, Pi calls your registered handlers.
```

### What You Can Register

- **Tools** — Functions the AI can call. You define the name, description, parameters (as a schema), and the execution logic. The AI sees the description and decides when to use the tool.
- **Commands** — Slash commands the human user types (e.g., `/theme`, `/tilldone`). You define the handler.
- **Shortcuts** — Keyboard shortcuts (e.g., Ctrl+X). You define what happens.
- **Event handlers** — Callbacks for lifecycle events: session start, tool calls, agent completion, user input, etc.

### The Context Object

When event handlers fire, they receive a `ctx` (ExtensionContext) object. This gives you access to:

- `ctx.ui` — all UI APIs (footer, widgets, status line, dialogs)
- `ctx.model` — information about the current AI model
- `ctx.cwd` — current working directory
- `ctx.sessionManager` — access to the conversation history
- `ctx.getContextUsage()` — how full the context window is
- `ctx.hasUI` — whether Pi is running in interactive (UI) mode

---

## 4. Extension Anatomy — Line by Line

Here is `minimal.ts` dissected completely:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";
```

**Line 1:** `import type` — imports the `ExtensionAPI` interface for TypeScript type checking only. This import vanishes at runtime. It comes from the `@mariozechner/pi-coding-agent` package, which Pi makes available.

**Line 2:** Imports a shared helper from `themeMap.ts` (in the same `extensions/` folder). This is a relative import, identified by the `./` prefix and the explicit `.ts` extension. When using jiti, the `.ts` extension in imports is required.

**Line 3:** Imports two utility functions from `@mariozechner/pi-tui`, Pi's terminal UI library. `truncateToWidth` cuts a string to a visible character width. `visibleWidth` measures the visible width of a string that may contain ANSI escape sequences (which take up zero visible width but add bytes to the string).

---

```typescript
export default function (pi: ExtensionAPI) {
```

The extension entry point. This is an anonymous function (no name after `function`) assigned as the module's default export. The parameter `pi` is typed as `ExtensionAPI` — this is how TypeScript gives you autocomplete and type checking on the API object. Pi calls this once at startup.

---

```typescript
    pi.on("session_start", async (_event, ctx) => {
```

Registers an event handler for the `session_start` event. `pi.on` takes the event name as a string and a handler function. The handler is `async` because it may need to `await` things. Parameters:
- `_event` — the event data. The leading underscore is a TypeScript/JavaScript convention meaning "I receive this parameter but do not use it."
- `ctx` — the `ExtensionContext` object for this session.

---

```typescript
        applyExtensionDefaults(import.meta.url, ctx);
```

Calls the shared helper from `themeMap.ts`. This does two things: sets the theme defined for this extension in the `THEME_MAP` (so `minimal.ts` gets the `synthwave` theme), and sets the terminal window title to `π - minimal`. `import.meta.url` is how ES modules get their own file path — equivalent to `__FILE__` in C or `getClass().getResource("")` in Java.

---

```typescript
        ctx.ui.setFooter((_tui, theme, _footerData) => ({
```

Registers a custom footer renderer. `setFooter` takes a factory function. Pi calls this factory once to create the footer object. The factory receives:
- `_tui` — the TUI (terminal UI) instance, used for requesting redraws
- `theme` — the current color theme object, used for applying colors
- `_footerData` — extra footer utilities (git branch, branch change events)

The `({` syntax is an arrow function returning an object literal. In JavaScript/TypeScript, when an arrow function returns an object literal directly, you wrap the braces in parentheses to distinguish them from a function body block.

---

```typescript
            dispose: () => {},
            invalidate() {},
```

These are required members of the footer object interface.
- `dispose` — called when the footer is torn down. Use it to clean up subscriptions or timers. Here it's a no-op (`() => {}` is an empty function).
- `invalidate` — called when the footer should clear any render cache and re-render on next tick. Here also a no-op because the `render` function does no caching.

---

```typescript
            render(width: number): string[] {
```

The render function. Pi calls this whenever it needs to draw the footer. `width` is the terminal column count. Returns an array of strings — one string per line. Each string may contain ANSI escape codes for color.

---

```typescript
                const model = ctx.model?.id || "no-model";
```

`ctx.model?.id` uses the **optional chaining operator** (`?.`). If `ctx.model` is `null` or `undefined`, the expression short-circuits to `undefined` instead of throwing a null pointer exception. The `||` then substitutes the fallback `"no-model"`. This is equivalent to:

```java
String model = ctx.getModel() != null ? ctx.getModel().getId() : "no-model";
```

---

```typescript
                const usage = ctx.getContextUsage();
                const pct = (usage && usage.percent !== null) ? usage.percent : 0;
```

Gets context window usage. `ctx.getContextUsage()` may return `null` if usage data is unavailable. `usage && usage.percent !== null` is a null guard: if `usage` is falsy (null/undefined), `pct` becomes `0`. Otherwise uses `usage.percent`. In TypeScript, `&&` short-circuits and returns the right operand if the left is truthy.

---

```typescript
                const filled = Math.round(pct / 10);
                const bar = "#".repeat(filled) + "-".repeat(10 - filled);
```

Builds a 10-block progress bar. `"#".repeat(n)` repeats the character `n` times — equivalent to `String.repeat(n)` in Java 11+.

---

```typescript
                const left = theme.fg("dim", ` ${model}`);
                const right = theme.fg("dim", `[${bar}] ${Math.round(pct)}% `);
```

Applies color using theme tokens. `theme.fg("dim", text)` wraps `text` in ANSI escape codes for the `dim` color token from the current theme. The backtick strings are **template literals** — equivalent to Java's `String.format()` or C's `sprintf`, but inline. `${expr}` interpolates the expression.

---

```typescript
                const pad = " ".repeat(Math.max(1, width - visibleWidth(left) - visibleWidth(right)));
```

Calculates padding to push the right content to the right edge. `visibleWidth` measures character width while ignoring ANSI escape codes. `Math.max(1, ...)` ensures at least one space even if the terminal is very narrow.

---

```typescript
                return [truncateToWidth(left + pad + right, width)];
```

Returns a single-element array (one footer line). `truncateToWidth` cuts the string to `width` visible characters, handling ANSI codes correctly so the cut does not split an escape sequence.

---

```typescript
            },
        }));
    });
}
```

Closing braces: end of `render` method, end of footer object, end of `setFooter` call, end of `session_start` handler, end of the extension function. The `));` closes `setFooter(` and `pi.on(` respectively.

---

## 5. The Extension API Reference

### 5.1 Registering Tools

Tools are functions the AI model can call. You define the schema; the AI reads your description and decides when to invoke the tool.

**Minimal structure:**

```typescript
pi.registerTool({
    name: "my_tool",           // snake_case identifier the AI uses
    label: "My Tool",          // human-readable label for the UI
    description: "...",        // CRITICAL: the AI reads this to know when to use the tool
    parameters: Type.Object({  // TypeBox schema — defines accepted parameters
        message: Type.String({ description: "The message to process" }),
        count: Type.Optional(Type.Number({ description: "How many times" })),
    }),

    async execute(toolCallId, params, signal, onUpdate, ctx) {
        // toolCallId: unique ID for this specific invocation
        // params: the validated parameters the AI passed in
        // signal: AbortSignal — check signal.aborted for cancellation
        // onUpdate: optional streaming callback for partial results
        // ctx: ExtensionContext — access to UI, session, etc.

        return {
            content: [{ type: "text", text: "Result text the AI sees" }],
            details: { anything: "you want for rendering" },
        };
    },

    renderCall(args, theme) {
        // Renders how the tool call appears in the UI before execution
        // args: the parameters as a plain object
        // Returns a TUI component (usually Text)
        return new Text(theme.fg("toolTitle", theme.bold("my_tool")), 0, 0);
    },

    renderResult(result, options, theme) {
        // Renders the tool result in the UI
        // result: what execute() returned
        // options: { expanded: boolean, isPartial: boolean }
        // expanded: user clicked to see full output
        // isPartial: streaming — result not yet final
        const text = result.content[0];
        return new Text(text?.type === "text" ? text.text : "", 0, 0);
    },
});
```

**TypeBox schemas** from `@sinclair/typebox`:

```typescript
import { Type } from "@sinclair/typebox";

Type.Object({ ... })           // object with named fields
Type.String({ description })   // string field
Type.Number({ description })   // number field
Type.Boolean({ description })  // boolean field
Type.Optional(Type.String())   // optional string
Type.Array(Type.String())      // array of strings
```

TypeBox generates JSON Schema from these calls. Pi validates the AI's parameters against this schema before calling your `execute` function.

**Full example from tilldone.ts:**

```typescript
import { Type } from "@sinclair/typebox";
import { StringEnum } from "@mariozechner/pi-ai";

const TillDoneParams = Type.Object({
    action: StringEnum(["new-list", "add", "toggle", "remove", "update", "list", "clear"] as const),
    text: Type.Optional(Type.String({ description: "Task text (for add/update)" })),
    id: Type.Optional(Type.Number({ description: "Task ID (for toggle/remove/update)" })),
});

pi.registerTool({
    name: "tilldone",
    label: "TillDone",
    description: "Manage your task list. Actions: new-list, add, toggle, remove, update, list, clear.",
    parameters: TillDoneParams,

    async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
        switch (params.action) {
            case "add": {
                // params.text is string | undefined here — TypeScript knows this
                if (!params.text) {
                    return {
                        content: [{ type: "text" as const, text: "Error: text required for add" }],
                        details: { error: "text required" },
                    };
                }
                // do the work...
                return {
                    content: [{ type: "text" as const, text: `Added: ${params.text}` }],
                    details: { action: "add" },
                };
            }
        }
    },

    renderCall(args, theme) {
        return new Text(
            theme.fg("toolTitle", theme.bold("tilldone ")) +
            theme.fg("muted", args.action),
            0, 0
        );
    },

    renderResult(result, { expanded }, theme) {
        const text = result.content[0];
        return new Text(
            text?.type === "text" ? theme.fg("success", text.text) : "",
            0, 0
        );
    },
});
```

Note the `as const` annotation on string literals in `content`. TypeScript's inference sometimes widens `"text"` to `string`, which would break the union type. `as const` keeps it as the literal type `"text"`.

**IMPORTANT:** Register tools at the **top level** of the extension function, not inside event handlers. Tools must be registered synchronously before any events fire.

```typescript
export default function (pi: ExtensionAPI) {
    // CORRECT: registered at top level
    pi.registerTool({ name: "my_tool", ... });

    pi.on("session_start", async (_event, ctx) => {
        // WRONG: do not register tools inside event handlers
        // pi.registerTool({ name: "my_tool", ... });  // BAD
    });
}
```

### 5.2 Registering Commands

Commands are slash commands the user can type in the Pi input (e.g., `/theme`, `/tilldone`).

```typescript
pi.registerCommand("my-command", {
    description: "What this command does",

    handler: async (args, ctx) => {
        // args: string — everything after /my-command
        // ctx: ExtensionContext
        const trimmed = args?.trim() ?? "";
        ctx.ui.notify(`You ran my-command with: ${trimmed}`, "info");
    },

    getArgumentCompletions: (prefix: string) => {
        // Optional: return autocomplete suggestions for the argument
        // prefix: what the user has typed so far after the command name
        return [
            { value: "option-one", label: "First option" },
            { value: "option-two", label: "Second option" },
        ].filter(item => item.value.startsWith(prefix));
    },
});
```

**Example from theme-cycler.ts — /theme command:**

```typescript
pi.registerCommand("theme", {
    description: "Select a theme: /theme or /theme <name>",
    handler: async (args, ctx) => {
        if (!ctx.hasUI) return;

        const arg = args.trim();

        if (arg) {
            // Direct switch: /theme dracula
            const result = ctx.ui.setTheme(arg);
            if (result.success) {
                ctx.ui.notify(`Theme: ${arg}`, "info");
            } else {
                ctx.ui.notify(`Theme not found: ${arg}`, "error");
            }
            return;
        }

        // No argument — show a selection dialog
        const themes = ctx.ui.getAllThemes();
        const items = themes.map(t => `${t.name} — ${t.path || "built-in"}`);
        const selected = await ctx.ui.select("Select Theme", items);
        if (!selected) return;

        const name = selected.split(/\s/)[0];
        ctx.ui.setTheme(name);
    },
});
```

### 5.3 Registering Keyboard Shortcuts

```typescript
pi.registerShortcut("ctrl+x", {
    description: "Cycle theme forward",
    handler: async (ctx) => {
        // ctx: ExtensionContext
        cycleTheme(ctx, 1);
    },
});

pi.registerShortcut("ctrl+q", {
    description: "Cycle theme backward",
    handler: async (ctx) => {
        cycleTheme(ctx, -1);
    },
});
```

The key string format is `"ctrl+key"`, `"alt+key"`, `"shift+key"`, or plain `"key"` for special keys like `"escape"`, `"enter"`, etc.

### 5.4 Event Handlers

`pi.on(eventType, handler)` registers a handler for a lifecycle event. Multiple handlers can be registered for the same event (across different stacked extensions). Handlers are called in registration order.

#### session_start

Fires once when a Pi session begins. Use it to set up UI and load config.

```typescript
pi.on("session_start", async (_event, ctx) => {
    // Set up footer, widgets, load config files, etc.
    applyExtensionDefaults(import.meta.url, ctx);
    ctx.ui.setStatus("ready", "My extension is ready");
});
```

The `session_start` event also fires when the user creates a new session with `/new`.

#### session_switch, session_fork, session_tree

These fire when the user navigates the session tree (e.g., branching the conversation, switching branches). Reconstruct any state you derive from session history.

```typescript
pi.on("session_switch", async (_event, ctx) => reconstructState(ctx));
pi.on("session_fork", async (_event, ctx) => reconstructState(ctx));
pi.on("session_tree", async (_event, ctx) => reconstructState(ctx));
```

#### tool_call

Fires before a tool is executed. Return `{ block: false }` to allow, or `{ block: true, reason: "..." }` to block the AI from using it. Use this to gate tool access.

```typescript
pi.on("tool_call", async (event, ctx) => {
    // event.toolName: which tool the AI wants to call
    // event.input: the parameters it passed

    if (event.toolName === "bash" && event.input.command.includes("rm -rf")) {
        return { block: true, reason: "Destructive commands are not allowed." };
    }

    return { block: false };
});
```

For type-safe access to tool input parameters, use `isToolCallEventType`:

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
    if (isToolCallEventType("bash", event)) {
        // TypeScript now knows event.input has a .command field
        const cmd = event.input.command;
    }
    if (isToolCallEventType("write", event)) {
        // TypeScript now knows event.input has .path and .content
        const path = event.input.path;
    }
    return { block: false };
});
```

#### before_agent_start

Fires before the AI model begins a new response turn. Return `{ systemPrompt: "..." }` to inject or replace the system prompt. This is how agent-team.ts and agent-chain.ts dynamically build the system prompt.

```typescript
pi.on("before_agent_start", async (event, ctx) => {
    // event.systemPrompt: the current system prompt
    return {
        systemPrompt: event.systemPrompt + "\n\nExtra instructions here.",
    };
    // Or to replace entirely:
    return { systemPrompt: "You are now a pirate." };
});
```

#### input

Fires when the user submits a message. Return `{ action: "continue" }` to let it proceed normally, or `{ action: "handled" }` to swallow it (block the AI from seeing it).

```typescript
pi.on("input", async (event, ctx) => {
    if (!purposeDeclared) {
        ctx.ui.notify("Declare your purpose first.", "warning");
        return { action: "handled" };  // user message is swallowed
    }
    return { action: "continue" };
});
```

#### agent_end

Fires when the AI finishes its response turn. Use it to trigger follow-up actions.

```typescript
pi.on("agent_end", async (_event, ctx) => {
    const incomplete = tasks.filter(t => t.status !== "done");
    if (incomplete.length > 0) {
        pi.sendMessage(
            { content: "You still have incomplete tasks!", display: true },
            { triggerTurn: true }
        );
    }
});
```

#### tool_execution_end

Fires after a tool finishes executing. Use it to track tool usage statistics.

```typescript
pi.on("tool_execution_end", async (event) => {
    // event.toolName: which tool was called
    // event.result: what it returned
    counts[event.toolName] = (counts[event.toolName] || 0) + 1;
});
```

#### session_shutdown

Fires when Pi is shutting down. Clean up timers, processes, or other resources.

```typescript
pi.on("session_shutdown", async () => {
    if (timer) {
        clearTimeout(timer);
    }
});
```

### 5.5 UI APIs

All UI APIs live on `ctx.ui`. They only work in interactive mode — check `ctx.hasUI` before calling them in headless/programmatic contexts.

#### ctx.ui.setFooter()

Replaces the default footer with a custom renderer.

```typescript
ctx.ui.setFooter((tui, theme, footerData) => {
    // tui: TUI instance — call tui.requestRender() to trigger a redraw
    // theme: current color theme
    // footerData.getGitBranch(): returns current git branch or empty string
    // footerData.onBranchChange(callback): subscribe to branch changes, returns unsubscribe fn

    const unsub = footerData.onBranchChange(() => tui.requestRender());

    return {
        dispose: unsub,          // called on teardown
        invalidate() {},         // called to clear render cache
        render(width: number): string[] {
            return [
                truncateToWidth(theme.fg("dim", " my footer line"), width)
            ];
        },
    };
});
```

The `render` function returns `string[]` — each element is one line. Use `truncateToWidth` to prevent overflow. Use `theme.fg(token, text)` to colorize.

#### ctx.ui.setWidget(key, factory, options)

Adds or removes a widget (a block of UI above or below the editor).

```typescript
// Add a widget below the editor
ctx.ui.setWidget("my-widget", (_tui, theme) => {
    return {
        render(width: number): string[] {
            return [theme.fg("accent", "  My widget content")];
        },
        invalidate() {},
    };
}, { placement: "belowEditor" });

// Remove a widget
ctx.ui.setWidget("my-widget", undefined);
```

The `key` string uniquely identifies the widget. Setting the same key again replaces the widget. Setting `undefined` removes it.

#### ctx.ui.setStatus(text, key)

Sets a status line entry. Multiple extensions can set different status entries — they are combined.

```typescript
ctx.ui.setStatus("ready", "My Extension: 5 items");
// key "ready" ensures only one status entry per key
```

#### ctx.ui.setTitle(text)

Sets the terminal window title.

```typescript
ctx.ui.setTitle("π - my-extension");
```

#### ctx.ui.setTheme(name)

Switches the active color theme. Returns `{ success: boolean, error?: string }`.

```typescript
const result = ctx.ui.setTheme("dracula");
if (!result.success) {
    console.error("Theme not found:", result.error);
}
```

#### ctx.ui.notify(text, level)

Shows a toast notification. Levels: `"info"` | `"success"` | `"warning"` | `"error"`.

```typescript
ctx.ui.notify("Operation complete", "success");
ctx.ui.notify("Something went wrong", "error");
```

#### ctx.ui.input(title, placeholder)

Shows a text input dialog. Returns the entered string, or `undefined` if cancelled. This is `async` — `await` it.

```typescript
const answer = await ctx.ui.input(
    "What is your name?",
    "e.g. Alice"
);
if (answer && answer.trim()) {
    // user typed something
}
```

#### ctx.ui.confirm(title, message, options)

Shows a yes/no confirmation dialog. Returns `true` if confirmed, `false` or `undefined` if cancelled. The `timeout` option auto-cancels after N milliseconds.

```typescript
const confirmed = await ctx.ui.confirm(
    "Delete all tasks?",
    `This will remove ${tasks.length} task(s). Continue?`,
    { timeout: 30000 }
);
if (!confirmed) return;
// proceed with deletion
```

#### ctx.ui.select(title, options)

Shows a selection list dialog. Returns the selected item string, or `undefined` if cancelled.

```typescript
const themes = ctx.ui.getAllThemes();
const items = themes.map(t => t.name);
const choice = await ctx.ui.select("Pick a theme", items);
if (!choice) return;
```

#### ctx.ui.custom(factory, options)

Shows a full-screen custom overlay. Returns a Promise that resolves with whatever value you pass to `done()`.

```typescript
await ctx.ui.custom<void>((_tui, theme, _keyboard, done) => {
    return {
        handleInput(data: string): void {
            if (data === "\x1b") done();  // Escape closes
        },
        render(width: number): string[] {
            return ["  My full-screen overlay"];
        },
        invalidate() {},
    };
});
```

### 5.6 Session and State

#### ctx.sessionManager.getBranch()

Returns an array of all session entries on the current conversation branch. Each entry has a `type` field: `"message"` (for AI/user/tool messages), `"input"` (for user input), or others.

```typescript
for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type !== "message") continue;
    const msg = entry.message;
    // msg.role: "user" | "assistant" | "toolCall" | "toolResult"
    // msg.toolName: set for toolCall and toolResult
    // msg.details: custom data stored with the tool result
}
```

#### pi.appendEntry(namespace, data)

Persists arbitrary data to the session history. This data survives session forks and can be reconstructed later.

```typescript
pi.appendEntry("damage-control-log", {
    tool: event.toolName,
    input: event.input,
    rule: violationReason,
    action: "blocked",
});
```

#### Reconstructing State from Session History

Because Pi sessions can be forked and branched, you cannot reliably keep state in JavaScript variables across branch switches. The correct pattern is to reconstruct state by replaying the session history.

The `tilldone` extension demonstrates this: every time the user switches branches, `reconstructState` is called. It walks all session entries looking for `toolResult` messages from the `tilldone` tool and reads the `details` field, which contains a snapshot of the task list at the time of that tool call.

```typescript
const reconstructState = (ctx: ExtensionContext) => {
    tasks = [];
    nextId = 1;

    for (const entry of ctx.sessionManager.getBranch()) {
        if (entry.type !== "message") continue;
        const msg = entry.message;
        if (msg.role !== "toolResult" || msg.toolName !== "tilldone") continue;

        // The most recent toolResult snapshot wins
        const details = msg.details as TillDoneDetails | undefined;
        if (details) {
            tasks = details.tasks;
            nextId = details.nextId;
        }
    }
};
```

The key insight is: your `execute` function returns `details` in every response. Pi stores this in the session. When branches switch, you replay history to rebuild state.

---

## 6. How to Create a New Extension

This tutorial walks you through creating a **word-count extension** that shows the total word count of the conversation in the footer.

### Step 1: Create the File

Create `/c/Users/simon/Downloads/pi-vs-claude-code-main/extensions/word-count.ts`.

### Step 2: Write the Basic Structure

```typescript
/**
 * Word Count — Shows total word count of the conversation in the footer.
 *
 * Usage: pi -e extensions/word-count.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";

export default function (pi: ExtensionAPI) {
    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        // Footer setup will go here
    });
}
```

### Step 3: Add the Footer

The footer needs to count words across all messages. We use `ctx.sessionManager.getBranch()` to walk the history, then request a redraw when the branch changes.

```typescript
export default function (pi: ExtensionAPI) {
    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);

        ctx.ui.setFooter((tui, theme, footerData) => {
            // Re-render whenever the branch changes (new messages added)
            const unsub = footerData.onBranchChange(() => tui.requestRender());

            return {
                dispose: unsub,
                invalidate() {},
                render(width: number): string[] {
                    // Count words in all messages
                    let wordCount = 0;
                    let messageCount = 0;

                    for (const entry of ctx.sessionManager.getBranch()) {
                        if (entry.type !== "message") continue;
                        const msg = entry.message;

                        // Count user and assistant text messages
                        if (msg.role === "user" || msg.role === "assistant") {
                            const text = typeof msg.content === "string"
                                ? msg.content
                                : "";
                            if (text.trim()) {
                                wordCount += text.trim().split(/\s+/).length;
                                messageCount++;
                            }
                        }
                    }

                    const left = theme.fg("dim", " Word Count");
                    const right = theme.fg("accent", `${wordCount} words`) +
                        theme.fg("dim", ` in ${messageCount} messages `);

                    const pad = " ".repeat(
                        Math.max(1, width - visibleWidth(left) - visibleWidth(right))
                    );

                    return [truncateToWidth(left + pad + right, width)];
                },
            };
        });
    });
}
```

### Step 4: Add it to themeMap.ts

Open `extensions/themeMap.ts` and add an entry to `THEME_MAP`:

```typescript
export const THEME_MAP: Record<string, string> = {
    // ... existing entries ...
    "word-count": "nord",   // or any theme you prefer
};
```

### Step 5: Run It

```bash
pi -e extensions/word-count.ts
```

Or stack it with minimal:

```bash
pi -e extensions/word-count.ts -e extensions/minimal.ts
```

### Step 6: Add a Just Recipe (Optional)

Add to `justfile`:

```
ext-word-count:
    pi -e extensions/word-count.ts
```

### Common Pitfalls When Creating Extensions

- **Forgetting `as const` on string literals in return objects** — TypeScript sometimes infers `string` instead of `"text"`, breaking the content type union. Use `{ type: "text" as const, text: "..." }`.
- **Using `ctx` from a stale closure** — Always use the `ctx` passed to the current event handler, not a captured reference from a different handler call.
- **Not checking `ctx.hasUI`** — UI calls throw or silently fail in `--mode json` (headless) runs. Always guard with `if (!ctx.hasUI) return;`.
- **Registering tools inside event handlers** — Tools must be registered at the top level, synchronously, when the extension function first runs.

---

## 7. How to Create a Multi-Agent Extension

This section walks through the pattern used by `agent-team.ts` to orchestrate multiple AI agents as subprocesses.

### The Core Concept

A multi-agent extension spawns Pi itself as child processes. The primary Pi (the one you launched) acts as a dispatcher. When it calls the `dispatch_agent` tool, the extension spawns `pi --mode json -p <prompt>` as a subprocess. The subprocess runs non-interactively, outputs JSONL events to stdout, and exits when done.

### Step 1: Define Agent Files

Agents are defined as `.md` files with YAML frontmatter. Store them in `.pi/agents/`.

```markdown
---
name: scout
description: Fast recon and codebase exploration
tools: read,grep,find,ls
---
You are a scout agent. Investigate the codebase quickly and report findings concisely.
Do NOT modify any files. Focus on structure, patterns, and key entry points.
```

The frontmatter fields:
- `name` — identifier used to dispatch to this agent
- `description` — shown to the dispatcher agent so it knows what to use this agent for
- `tools` — comma-separated list of built-in tools this agent can access
- Body — the system prompt injected into the subagent

### Step 2: Parse Agent Files on session_start

```typescript
function parseAgentFile(filePath: string): AgentDef | null {
    try {
        const raw = readFileSync(filePath, "utf-8");
        // Match ---\n<frontmatter>\n---\n<body>
        const match = raw.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
        if (!match) return null;

        const frontmatter: Record<string, string> = {};
        for (const line of match[1].split("\n")) {
            const idx = line.indexOf(":");
            if (idx > 0) {
                frontmatter[line.slice(0, idx).trim()] = line.slice(idx + 1).trim();
            }
        }

        if (!frontmatter.name) return null;

        return {
            name: frontmatter.name,
            description: frontmatter.description || "",
            tools: frontmatter.tools || "read,grep,find,ls",
            systemPrompt: match[2].trim(),
        };
    } catch {
        return null;
    }
}
```

### Step 3: Register a Dispatch Tool at the Top Level

```typescript
export default function (pi: ExtensionAPI) {
    // State — lives at extension scope, shared across all event handlers
    const agentStates: Map<string, AgentState> = new Map();
    let widgetCtx: any;

    // Register the tool OUTSIDE any event handler
    pi.registerTool({
        name: "dispatch_agent",
        label: "Dispatch Agent",
        description: "Dispatch a task to a specialist agent.",
        parameters: Type.Object({
            agent: Type.String({ description: "Agent name" }),
            task: Type.String({ description: "Task description" }),
        }),

        async execute(_id, params, _signal, onUpdate, ctx) {
            const result = await dispatchAgent(params.agent, params.task, ctx);
            return {
                content: [{ type: "text" as const, text: result.output }],
                details: { agent: params.agent, status: "done" },
            };
        },
        // ... renderCall, renderResult
    });

    // ... session_start, before_agent_start, etc.
}
```

### Step 4: Spawn Pi Subprocesses

```typescript
import { spawn } from "child_process";

function dispatchAgent(
    agentName: string,
    task: string,
    ctx: any,
): Promise<{ output: string; exitCode: number }> {
    const model = ctx.model
        ? `${ctx.model.provider}/${ctx.model.id}`
        : "openrouter/google/gemini-3-flash-preview";

    const args = [
        "--mode", "json",         // output JSONL events to stdout
        "-p",                     // non-interactive (print) mode
        "--no-extensions",        // do not load any extensions in subagent
        "--model", model,         // use same model as parent
        "--tools", "read,bash,grep,find,ls",  // tools available to this agent
        "--thinking", "off",      // disable extended thinking for speed
        "--append-system-prompt", agentSystemPrompt,
        task,                     // the prompt
    ];

    return new Promise((resolve) => {
        const proc = spawn("pi", args, {
            stdio: ["ignore", "pipe", "pipe"],  // no stdin, capture stdout/stderr
            env: { ...process.env },
        });

        const textChunks: string[] = [];
        let buffer = "";

        proc.stdout!.setEncoding("utf-8");
        proc.stdout!.on("data", (chunk: string) => {
            buffer += chunk;
            const lines = buffer.split("\n");
            buffer = lines.pop() || "";  // keep incomplete line in buffer

            for (const line of lines) {
                if (!line.trim()) continue;
                try {
                    const event = JSON.parse(line);

                    if (event.type === "message_update") {
                        const delta = event.assistantMessageEvent;
                        if (delta?.type === "text_delta") {
                            textChunks.push(delta.delta || "");
                            updateWidget();  // refresh the UI card for this agent
                        }
                    } else if (event.type === "tool_execution_start") {
                        toolCount++;
                        updateWidget();
                    }
                } catch {}  // silently skip malformed lines
            }
        });

        proc.stderr!.setEncoding("utf-8");
        proc.stderr!.on("data", () => {});  // discard stderr

        proc.on("close", (code) => {
            // flush remaining buffer
            if (buffer.trim()) {
                try {
                    const event = JSON.parse(buffer);
                    if (event.type === "message_update") {
                        const delta = event.assistantMessageEvent;
                        if (delta?.type === "text_delta") textChunks.push(delta.delta || "");
                    }
                } catch {}
            }

            resolve({
                output: textChunks.join(""),
                exitCode: code ?? 1,
            });
        });

        proc.on("error", (err) => {
            resolve({ output: `Error: ${err.message}`, exitCode: 1 });
        });
    });
}
```

### Step 5: Override System Prompt in before_agent_start

The dispatcher needs to know what agents exist. Build this dynamically in `before_agent_start`:

```typescript
pi.on("before_agent_start", async (_event, _ctx) => {
    const catalog = Array.from(agentStates.values())
        .map(s => `### ${s.def.name}\n${s.def.description}\n**Tools:** ${s.def.tools}`)
        .join("\n\n");

    return {
        systemPrompt: `You are a dispatcher. You have NO direct codebase tools.
You MUST delegate all work using dispatch_agent.

## Available Agents

${catalog}`,
    };
});
```

### Step 6: Update the Widget in Real Time

As the subprocess sends events, update the widget to show progress:

```typescript
function updateWidget() {
    if (!widgetCtx) return;
    widgetCtx.ui.setWidget("agent-grid", (_tui: any, theme: any) => {
        return {
            render(width: number): string[] {
                const lines: string[] = [];
                for (const [, state] of agentStates) {
                    const icon = state.status === "running" ? "●" : state.status === "done" ? "✓" : "○";
                    const color = state.status === "running" ? "accent" : state.status === "done" ? "success" : "dim";
                    lines.push(theme.fg(color, `${icon} ${state.def.name}: ${state.lastWork || state.def.description}`));
                }
                return lines.length > 0 ? lines : [theme.fg("dim", "No agents dispatched yet")];
            },
            invalidate() {},
        };
    });
}
```

### Step 7: Lock Down Primary Agent's Tools

When using a pure dispatcher pattern, prevent the primary agent from using codebase tools directly:

```typescript
pi.on("session_start", async (_event, ctx) => {
    // ...
    pi.setActiveTools(["dispatch_agent"]);
});
```

---

## 8. How to Add Safety Rules

The `damage-control.ts` extension provides rule-based tool interception. Rules are defined in `.pi/damage-control-rules.yaml`.

### Rule File Format

Create `.pi/damage-control-rules.yaml` in your project root:

```yaml
# Block bash commands matching these patterns
bashToolPatterns:
  - pattern: "rm -rf /"
    reason: "Deleting the root filesystem is not allowed"
  - pattern: "DROP TABLE"
    reason: "Database drops require manual confirmation"
    ask: true   # ask: true shows a confirm dialog instead of hard-blocking

# Block read/write/grep access to these paths entirely
zeroAccessPaths:
  - ~/.ssh/
  - ~/.aws/
  - /etc/passwd

# Allow reads but block writes/edits to these paths
readOnlyPaths:
  - package.json
  - package-lock.json

# Allow reads and writes but block deletes/moves
noDeletePaths:
  - src/core/
  - migrations/
```

### How Tool Call Interception Works

The extension registers a `tool_call` handler that runs before every tool execution:

```typescript
pi.on("tool_call", async (event, ctx) => {
    // Check all rules...
    const violation = findViolation(event, ctx);

    if (!violation) {
        return { block: false };  // allow the tool call
    }

    if (violation.shouldAsk) {
        // Show confirmation dialog
        const confirmed = await ctx.ui.confirm(
            "Dangerous command detected",
            `${violation.reason}\n\nProceed?`,
            { timeout: 30000 }
        );

        if (confirmed) {
            return { block: false };
        }
    }

    // Hard block
    ctx.abort();  // cancel the current agent turn
    return {
        block: true,
        reason: `BLOCKED: ${violation.reason}. Do NOT retry with alternatives.`,
    };
});
```

### Path Matching

Paths in rules support:
- `~` prefix — expands to the user's home directory
- `*` wildcard — matches any segment
- Trailing `/` — matches the directory and everything under it
- Substring matching — if the path appears anywhere in the tool's path argument

### The ctx.abort() Call

`ctx.abort()` cancels the current agent turn. Without it, the AI would receive the block message and immediately try again. `abort()` stops the entire agent response, so the user gets control back.

---

## 9. How Themes Work

### Theme Files

Themes are JSON files in `.pi/themes/`. Each file defines a set of named variables (color hex values) and then maps semantic token names to those variables.

**Example: `.pi/themes/synthwave.json`**

```json
{
  "$schema": "https://raw.githubusercontent.com/badlogic/pi-mono/main/.../theme-schema.json",
  "name": "synthwave",
  "vars": {
    "bg":     "#262335",
    "cyan":   "#36f9f6",
    "green":  "#72f1b8",
    "pink":   "#ff7edb"
  },
  "colors": {
    "accent":    "cyan",
    "success":   "green",
    "warning":   "orange",
    "error":     "red",
    "dim":       "comment",
    "muted":     "comment",
    "border":    "pink",
    "toolTitle": "orange"
  }
}
```

The `vars` section is a palette — raw color hex values with meaningful names. The `colors` section maps semantic tokens to palette variable names. This two-level system means you can change all "accent" colors across the UI by editing a single `vars` entry.

### The 51 Color Tokens

Tokens you will use in extensions:

| Token | Meaning |
|---|---|
| `accent` | Highlights, interactive elements, active state |
| `success` | Completed tasks, done status, positive outcomes |
| `warning` | Caution, pending, in-progress |
| `error` | Failures, blocked, errors |
| `dim` | Subdued text, secondary information |
| `muted` | Slightly more visible than dim |
| `text` | Primary foreground text |
| `border` | Box borders |
| `borderMuted` | Subtle box borders |
| `borderAccent` | Highlighted borders |
| `toolTitle` | Tool name in tool call renders |
| `toolOutput` | Tool result text |

Tokens for syntax highlighting, markdown rendering, and thinking levels also exist but are rarely used in extension code.

### Using Theme Tokens in Code

```typescript
// In render functions and renderCall/renderResult:
theme.fg("accent", "some text")    // foreground color
theme.bold("some text")            // bold
theme.fg("success", theme.bold("done"))  // bold green

// Raw ANSI escape codes (avoid if possible — breaks portability):
"\x1b[38;2;255;126;219m" + text + "\x1b[39m"  // RGB foreground
"\x1b[48;2;74;30;106m" + text + "\x1b[49m"    // RGB background
```

Prefer `theme.fg()` over raw ANSI codes. Raw codes are sometimes necessary for custom background fills (as in `purpose-gate.ts` and `pi-pi.ts`) when the theme API does not expose the color you need.

### The themeMap.ts Pattern

Every extension that wants an automatic theme assignment:

1. Adds an entry to `THEME_MAP` in `extensions/themeMap.ts`
2. Calls `applyExtensionDefaults(import.meta.url, ctx)` in its `session_start` handler

`applyExtensionDefaults` does two things:
- Calls `ctx.ui.setTheme(themeName)` for the mapped theme
- Calls `ctx.ui.setTitle("π - <extension-name>")` after a 150ms delay (to override Pi's own startup title)

### Stacking Without Theme Conflicts

When multiple extensions are stacked (`pi -e ext1.ts -e ext2.ts`), all `session_start` handlers fire. Each extension may try to set a theme. Without coordination, the last one wins — which would always be the last extension listed.

The solution in `themeMap.ts` is `primaryExtensionName()`: it reads `process.argv` to find the first `-e` flag. Only the primary (first-listed) extension is allowed to set the theme. All others skip theme application but do not error.

```typescript
// themeMap.ts — simplified
function primaryExtensionName(): string | null {
    for (let i = 0; i < process.argv.length - 1; i++) {
        if (process.argv[i] === "-e") {
            return basename(process.argv[i + 1]).replace(/\.[^.]+$/, "");
        }
    }
    return null;
}

export function applyExtensionTheme(fileUrl: string, ctx: ExtensionContext): boolean {
    const name = extensionName(fileUrl);
    const primary = primaryExtensionName();
    if (primary && primary !== name) return true;  // skip — not primary
    return ctx.ui.setTheme(THEME_MAP[name] ?? "synthwave").success;
}
```

---

## 10. Testing and Debugging

### Running with Just Recipes

Use `just` recipes for consistent launch with the right extension stack:

```bash
just ext-minimal          # minimal extension
just ext-tilldone         # tilldone extension
just ext-damage-control   # damage-control with safety rules
```

### Stacking Extensions for Testing

You can combine any extensions at the command line:

```bash
# Test your new extension alongside the theme cycler and minimal footer
pi -e extensions/my-extension.ts -e extensions/theme-cycler.ts -e extensions/minimal.ts
```

### Using Tool Counter to Observe Tool Usage

Stack `tool-counter-widget.ts` to see what tools the AI is calling and how often:

```bash
pi -e extensions/my-extension.ts -e extensions/tool-counter-widget.ts
```

The widget shows a live count per tool name, updating as calls complete.

### Console.log Debugging

`console.log()` and `console.error()` output goes to Pi's stderr. You will not see it in the main UI by default, but you can redirect or watch the terminal that launched Pi.

```typescript
pi.on("session_start", async (_event, ctx) => {
    console.log("session_start fired, cwd:", ctx.cwd);
    console.log("model:", ctx.model?.id);
});
```

For production code, use `ctx.ui.notify()` instead of console.log — it shows in the Pi notification area.

### The --mode json Flag for Programmatic Testing

Running Pi with `--mode json -p <prompt>` makes it output newline-delimited JSON (JSONL) to stdout and exit when done. This is how agent-team and agent-chain spawn subagents.

You can use this for manual testing:

```bash
pi --mode json -p "List the files in this directory" --no-extensions
```

Each line of output is a JSON event:

```json
{"type": "message_update", "assistantMessageEvent": {"type": "text_delta", "delta": "Here "}}
{"type": "message_update", "assistantMessageEvent": {"type": "text_delta", "delta": "are the files:\n"}}
{"type": "tool_execution_start", "toolName": "ls", "toolCallId": "abc123"}
{"type": "agent_end", "messages": [...]}
```

### Testing State Reconstruction

For extensions that reconstruct state from session history (like `tilldone`), test by:
1. Starting a session
2. Having the AI use the tool several times
3. Using Pi's `/fork` command to create a branch
4. Switching back to the original branch with `/tree`
5. Verifying the state matches what you expect for that branch point

---

## 11. Code Style and Conventions

### Extensions Are Standalone Files

Each extension is a single `.ts` file. There is no separate test file, no separate types file. Keep everything in one place unless the file becomes unwieldy (in which case, a shared helper like `themeMap.ts` is appropriate).

### Register Tools at Top Level

```typescript
export default function (pi: ExtensionAPI) {
    // ALL tool registrations happen here, synchronously, at the top
    pi.registerTool({ name: "my_tool", ... });
    pi.registerTool({ name: "other_tool", ... });

    // Event handlers come after
    pi.on("session_start", async (_event, ctx) => { ... });
}
```

Registering tools inside event handlers fails. Pi collects tool registrations during the synchronous setup phase.

### Type-Safe Tool Call Event Narrowing

When handling `tool_call` events and checking the tool name, use `isToolCallEventType`:

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
    // Without isToolCallEventType: event.input is unknown
    // With it: TypeScript knows the exact shape of event.input
    if (isToolCallEventType("bash", event)) {
        const cmd = event.input.command;  // string — TypeScript confirmed
    }
    if (isToolCallEventType("write", event)) {
        const path = event.input.path;   // string — TypeScript confirmed
    }
    return { block: false };
});
```

### Use TypeBox for Tool Parameters

Always use `@sinclair/typebox` for tool parameter schemas. Never use raw JSON Schema objects. TypeBox gives you TypeScript inference of parameter types in your `execute` function.

### Theme Token Usage

| Use this token | For this purpose |
|---|---|
| `success` | Completed state, positive confirmations, "done" text |
| `accent` | Highlights, interactive labels, active agent names |
| `warning` | Pending state, in-progress indicators |
| `dim` | Secondary text, subdued labels, borders |
| `muted` | Readable but not prominent — task text, descriptions |
| `error` | Failures, blocked states |
| `toolTitle` | Tool names in renderCall functions |

### Footer Return Format

`render(width: number): string[]` — each element of the array is one rendered line of the footer. Fewer lines means a smaller footer. Use `truncateToWidth` on every line. Use `visibleWidth` when calculating padding.

### Widget Component Patterns

```typescript
// Simple: use Text for anything that fits in a Text node
const text = new Text("", paddingTop, paddingLeft);
return {
    render(width: number): string[] {
        text.setText("content here");
        return text.render(width);
    },
    invalidate() { text.invalidate(); },
};

// With borders: use Container + DynamicBorder
const container = new Container();
const borderFn = (s: string) => theme.fg("dim", s);
container.addChild(new DynamicBorder(borderFn));
const content = new Text("", 1, 0);
container.addChild(content);
container.addChild(new DynamicBorder(borderFn));
return {
    render(width: number): string[] {
        content.setText("content here");
        return container.render(width);
    },
    invalidate() { container.invalidate(); },
};
```

---

## 12. Common Patterns

### Pattern: Reconstructing State from Session History

Used in: `tilldone.ts`

When: Any extension that maintains state that can be derived from tool call history.

Why: Pi sessions can be forked and branched. JavaScript variable state does not survive branch switches. The session history does.

```typescript
// Store snapshots in tool result details
async execute(_id, params, _signal, _onUpdate, ctx) {
    // ... do the work, update in-memory state ...
    return {
        content: [{ type: "text" as const, text: "done" }],
        details: { tasks: [...tasks], nextId },  // snapshot goes here
    };
}

// Reconstruct from history on branch change
const reconstructState = (ctx: ExtensionContext) => {
    tasks = [];
    for (const entry of ctx.sessionManager.getBranch()) {
        if (entry.type !== "message") continue;
        if (entry.message.role !== "toolResult") continue;
        if (entry.message.toolName !== "my_tool") continue;
        const snapshot = entry.message.details as MyDetails | undefined;
        if (snapshot) { tasks = snapshot.tasks; }
    }
};

pi.on("session_start", async (_event, ctx) => { reconstructState(ctx); });
pi.on("session_switch", async (_event, ctx) => { reconstructState(ctx); });
pi.on("session_fork", async (_event, ctx) => { reconstructState(ctx); });
pi.on("session_tree", async (_event, ctx) => { reconstructState(ctx); });
```

### Pattern: Blocking Tool Execution Until Conditions Met

Used in: `tilldone.ts`, `purpose-gate.ts`

When: You want to force the AI to do something before it can proceed.

```typescript
pi.on("tool_call", async (event, _ctx) => {
    if (event.toolName === "my_setup_tool") return { block: false };

    if (!setupComplete) {
        return {
            block: true,
            reason: "You MUST call my_setup_tool first before using any other tools.",
        };
    }

    return { block: false };
});
```

The reason string is sent to the AI model — make it imperative and explicit to prevent the AI from trying workarounds.

### Pattern: Auto-Nudge on agent_end

Used in: `tilldone.ts`

When: You want the AI to automatically continue working if conditions are not met.

```typescript
let nudgedThisCycle = false;

pi.on("agent_end", async (_event, _ctx) => {
    if (conditionMet || nudgedThisCycle) return;
    nudgedThisCycle = true;

    pi.sendMessage(
        {
            content: "You still have incomplete work. Continue or mark it done.",
            display: true,
        },
        { triggerTurn: true },  // triggers a new agent turn immediately
    );
});

pi.on("input", async () => {
    nudgedThisCycle = false;  // reset when user sends a new message
    return { action: "continue" as const };
});
```

`nudgedThisCycle` prevents infinite nudge loops: once the AI is nudged, it will not be nudged again until the user sends a new message.

### Pattern: Fire-and-Forget Subagent Spawning

Used in: `subagent-widget.ts`

When: You want to spawn a subagent and let it run in the background while the primary agent continues.

```typescript
// In a tool's execute function:
async execute(_id, params, _signal, _onUpdate, ctx) {
    const state = createState(params.task);
    agents.set(state.id, state);
    updateWidgets();  // show the widget immediately

    // Fire-and-forget: start the subprocess, do not await it
    spawnAgent(state, params.task, ctx);

    // Return immediately — agent is running in background
    return {
        content: [{ type: "text" as const, text: `Subagent #${state.id} started.` }],
    };
}

// When the subprocess finishes, deliver results as a follow-up:
proc.on("close", (code) => {
    pi.sendMessage(
        { content: `Subagent finished: ${output}`, display: true },
        { deliverAs: "followUp", triggerTurn: true }
    );
    resolve();
});
```

### Pattern: Dispatcher with No Direct Tools

Used in: `agent-team.ts`

When: The primary agent should only coordinate, never act directly.

```typescript
pi.on("session_start", async (_event, ctx) => {
    loadAgents(ctx.cwd);
    // Restrict primary agent to only the dispatch tool
    pi.setActiveTools(["dispatch_agent"]);
});

pi.on("before_agent_start", async () => {
    return { systemPrompt: buildDispatcherPrompt() };
});
```

Combining `setActiveTools` with a clear dispatcher system prompt ensures the primary agent cannot accidentally use file tools.

### Pattern: Sequential Pipeline with Variable Substitution

Used in: `agent-chain.ts`

When: You have a fixed sequence of agents where each step's output feeds the next.

```typescript
async function runChain(task: string): Promise<string> {
    let input = task;
    const original = task;

    for (const step of activeChain.steps) {
        const prompt = step.prompt
            .replace(/\$INPUT/g, input)      // current step's input
            .replace(/\$ORIGINAL/g, original);  // always the user's first message

        const result = await runAgent(step.agent, prompt);
        if (result.exitCode !== 0) throw new Error(`Step failed: ${result.output}`);

        input = result.output;  // output becomes next step's input
    }

    return input;  // final output
}
```

### Pattern: Parallel Research with Promise.allSettled

Used in: `pi-pi.ts`

When: Multiple independent tasks can run simultaneously and you want all results even if some fail.

```typescript
const settled = await Promise.allSettled(
    queries.map(({ expert, question }) => queryExpert(expert, question, ctx))
);

const results = settled.map((s, i) =>
    s.status === "fulfilled"
        ? s.value
        : { expert: queries[i].expert, output: `Error: ${s.reason}`, status: "error" }
);
```

`Promise.allSettled` (unlike `Promise.all`) never throws. Each element of the result is either `{ status: "fulfilled", value: T }` or `{ status: "rejected", reason: unknown }`. This means one failed expert does not discard results from the others.

### Pattern: Frontmatter Parsing for Agent Definitions

Used in: `agent-team.ts`, `agent-chain.ts`, `pi-pi.ts`

When: Agent definitions live in `.md` files with YAML frontmatter.

The regex `raw.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/)` matches:
- `---\n` — opening fence
- `([\s\S]*?)` — non-greedy capture of frontmatter (the `?` makes it non-greedy)
- `\n---\n` — closing fence
- `([\s\S]*)` — rest of file (the system prompt body)

`[\s\S]*` matches any character including newlines (unlike `.` which does not match `\n` by default in JavaScript).

### Pattern: applyExtensionDefaults for Consistent Theme/Title

Used in: every extension

```typescript
// At the top of session_start, before doing anything else:
pi.on("session_start", async (_event, ctx) => {
    applyExtensionDefaults(import.meta.url, ctx);
    // ... rest of setup
});
```

This single call handles theme selection (from THEME_MAP) and terminal title setting. Always call it first, before any other UI setup, so the theme is set before you start rendering themed components.

---

## 13. Glossary

**extension** — A `.ts` file that exports a default function. Pi loads it at startup using jiti. Your extension customizes Pi's behavior for a session.

**tool** — A function the AI model can invoke. You define the name, description (which the AI reads), parameter schema (TypeBox), and execute function. The AI calls tools to do work (read files, run commands, etc.).

**command** — A slash command typed by the human user in the Pi input box. Prefixed with `/`. You register commands with `pi.registerCommand()`.

**shortcut** — A keyboard shortcut handled by an extension. Registered with `pi.registerShortcut()`.

**event** — A lifecycle moment Pi broadcasts to registered handlers. Examples: `session_start`, `tool_call`, `agent_end`, `input`.

**handler** — An async function registered with `pi.on()` that is called when an event fires.

**session** — A Pi conversation including all messages, tool calls, and branches. Sessions are saved as JSON files.

**branch** — A linear path through the session tree from root to a leaf. When you fork a conversation, two branches diverge. `ctx.sessionManager.getBranch()` returns the entries on the current branch.

**fork** — A diverging point in the session tree, created when the user uses `/fork` or navigates the tree. Like a git branch.

**widget** — A rectangular block of terminal UI rendered above or below the editor. Registered with `ctx.ui.setWidget()`.

**footer** — The bottom bar of the Pi UI. Extensions can replace it with `ctx.ui.setFooter()`. Returns `string[]` from its `render` function — one string per line.

**status line** — A line in the Pi UI that shows short status text. Multiple extensions can contribute entries via `ctx.ui.setStatus(key, text)`.

**overlay** — A full-screen modal UI layer shown with `ctx.ui.custom()`. Blocks the rest of the UI until dismissed.

**theme** — A JSON file in `.pi/themes/` defining a color palette and semantic token mappings. Applied with `ctx.ui.setTheme(name)`.

**token (color)** — A named semantic color in a theme, such as `"accent"`, `"success"`, `"dim"`. Extensions use token names rather than hex values so the same code works with any theme.

**token (LLM)** — A unit of text processed by an AI model. Roughly 3/4 of a word on average. Context windows are measured in tokens.

**context window** — The maximum number of tokens an AI model can process in one conversation. `ctx.getContextUsage()` returns how full it is.

**system prompt** — Instructions sent to the AI at the start of a conversation, before any user messages. Extensions can inject additional system prompt content in `before_agent_start`.

**agent** — In this project: an AI model instance with a specific system prompt and tool set. Can refer to the primary Pi agent (the interactive session) or a subagent (a subprocess spawned by an extension).

**subagent** — A Pi subprocess spawned by an extension via `spawn("pi", ["--mode", "json", ...])`. Runs non-interactively, outputs JSONL, and exits when done.

**dispatcher** — An agent whose only tool is dispatching to subagents. It has no direct codebase tools. Used in `agent-team.ts`.

**pipeline** — A fixed sequence of agents where each step's output becomes the next step's input. Used in `agent-chain.ts`.

**chain** — Synonym for pipeline in this project. Defined in `.pi/agents/agent-chain.yaml`.

**TypeBox** — The library (`@sinclair/typebox`) used to define JSON schemas for tool parameters. `Type.Object({ ... })`, `Type.String()`, `Type.Number()`, etc.

**jiti** — Just-In-Time TypeScript compiler. Pi uses it to load `.ts` extension files without a build step. You edit a `.ts` file, restart Pi, and your changes are live.

**JSONL** — JSON Lines. One JSON object per line, newline-delimited. The format Pi uses for `--mode json` output. Extensions parse this to read events from subprocesses.

**frontmatter** — YAML metadata at the top of a `.md` file, delimited by `---` fences. Agent definition files use frontmatter for `name`, `description`, and `tools` fields.

**ExtensionAPI** — The object passed to your extension's default function. Has methods: `on()`, `registerTool()`, `registerCommand()`, `registerShortcut()`, `sendMessage()`, `appendEntry()`, `setActiveTools()`.

**ExtensionContext** — The `ctx` object passed to event handlers. Has: `ui`, `model`, `cwd`, `sessionManager`, `getContextUsage()`, `hasUI`, `abort()`.

**isToolCallEventType** — A type guard function from `@mariozechner/pi-coding-agent` that narrows a `tool_call` event to a specific tool type, giving TypeScript access to the typed `.input` fields.

**applyExtensionDefaults** — A helper in `themeMap.ts` that sets the mapped theme and terminal title for an extension. Call it first in every `session_start` handler.

**primaryExtensionName** — The extension named first in the `-e` flags on the Pi command line. Determines which extension's theme wins when multiple extensions are stacked.
