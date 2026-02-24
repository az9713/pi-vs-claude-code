# STUDY PLAN: Pi vs Claude Code — From Zero to Hero

A systematic, zero-to-hero guide for engineers with C/C++/Java experience who are new
to TypeScript, terminal UI, AI agents, and multi-agent orchestration. No time constraint.
The only requirement is mastery.

---

## How to Use This Plan

Each phase builds directly on the previous one. Do not skip phases. The exercises are
mandatory — reading without doing produces shallow understanding. Self-assessment
checkpoints tell you whether you have genuinely internalized a phase before moving on.

Reference files use absolute paths from the project root. Line counts are exact so you
know what you are getting into before opening a file.

---

## Phase 0: Foundation — Understanding the Landscape

**Goal:** Build the mental models that make everything else learnable. This phase is
theory-heavy by design. You cannot effectively read unfamiliar code without a working
model of what the code is trying to accomplish.

---

### 0.1 What Are AI Coding Agents?

**Theory to understand:**

An LLM (Large Language Model) is a program that predicts the next token given all
previous tokens. A token is roughly 0.75 words. The model has no persistent memory
between conversations — its entire "working memory" is the context window, a fixed-size
buffer of past tokens (often 128K to 200K tokens).

A "coding agent" is an LLM connected to tools. The agent loop runs like this:

```
User sends prompt
  → LLM receives prompt + system prompt + conversation history
  → LLM either responds with text OR decides to call a tool
  → If tool call: tool executes, result appended to context
  → Loop repeats until LLM produces a final text response
```

The four default Pi tools are: `read` (read file contents), `write` (create/overwrite
file), `edit` (surgical find-replace), `bash` (run shell command). With just these four
tools an agent can read your entire codebase, write new files, run tests, fix bugs, and
commit changes.

**Key concepts for C/Java developers:**

- Context window = fixed-size ring buffer. When it fills, oldest entries are summarized.
- System prompt = a string prepended to every conversation, like a global `#define` that
  shapes all behavior.
- Tool call = the LLM emitting a structured JSON blob instead of text. The harness parses
  it, runs the tool, and feeds the result back as a new message.
- Tokens in/out = the "clock cycles" you are being billed for.

**Why this matters:** Every extension in this project hooks into this loop. Understanding
the loop tells you exactly where each hook fires and why.

---

### 0.2 Pi Coding Agent vs Claude Code

**Files to read:**

- `COMPARISON.md` (the entire file, ~244 lines)
- `README.md` (skim for overview, focus on the Extensions table)

**Key differences to internalize:**

| Dimension | Claude Code | Pi |
|-----------|------------|-----|
| Source | Closed, proprietary | Open source (MIT) |
| System prompt | ~10,000 tokens of rules | ~200 tokens — trusts the model |
| Safety | Deny-first, 5 permission modes | YOLO by default |
| Extensions | Shell hooks (external processes) | TypeScript in-process |
| Sub-agents | Built into Task tool | Extension-provided via subprocess |
| Model support | Claude family only | 324 models, 20+ providers |

**The critical insight:** Pi's extension system runs TypeScript *in the same process* as
Pi. This means extensions have direct access to the session state, can block any event
synchronously, can render arbitrary terminal UI, and can register tools that the LLM sees
as first-class tools. Claude Code hooks are external shell processes — they get JSON on
stdin and respond via stdout. Powerful, but looser.

**Exercise:** Read COMPARISON.md's "Hooks & Event System" section. For each Claude Code
hook event, find its Pi equivalent in the event table. Note which events exist only in
Pi. Answer in writing: why does Pi have `before_agent_start` but Claude Code does not?

---

### 0.3 TypeScript for C/Java Developers

**Theory: Language mental model**

TypeScript is JavaScript with a type system. JavaScript runs on the V8 engine (same
engine in Chrome). Bun replaces Node.js as the runtime — same V8 engine, faster startup.

Key syntax for C/Java developers:

```typescript
// Variables (prefer const over let, never var)
const name: string = "pi";           // like final String name = "pi";
let count: number = 0;               // like int count = 0;

// Functions — two equivalent forms
function add(a: number, b: number): number { return a + b; }
const add = (a: number, b: number): number => a + b;  // arrow function (lambda)

// Async functions (JavaScript's answer to threads)
async function fetchData(): Promise<string> {
    const result = await someAsyncOperation();  // like await in C# / CompletableFuture
    return result;
}

// Interfaces (like Java interfaces but structural, not nominal)
interface User {
    name: string;
    age?: number;    // ? = optional field
}

// Type aliases (like typedef in C)
type Status = "idle" | "running" | "done";  // union type = enum-like

// Destructuring (like structured bindings in C++17)
const { name, age } = user;                // extract fields from object
const [first, ...rest] = array;            // extract from array

// Optional chaining (like safe navigation in Kotlin/C#)
const id = ctx.model?.id ?? "unknown";    // short-circuits if null/undefined

// Template literals (like printf but for strings)
const msg = `Model: ${name}, usage: ${pct}%`;

// Modules — every .ts file is a module
import { something } from "./other-file.ts";   // like #include / import
export function myFunction() { ... }           // like public in Java
export default function (pi: ExtensionAPI) {}  // default export — one per file
```

**Imports from packages (like Maven dependencies):**

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
// ^ "import type" = only the type definition, erased at runtime (zero overhead)

import { truncateToWidth } from "@mariozechner/pi-tui";
// ^ regular value import — the function is actually available at runtime
```

**Files to read for TypeScript patterns:**

- `extensions/pure-focus.ts` (25 lines) — the absolute minimum structure
- `extensions/minimal.ts` (34 lines) — types, arrow functions, template literals

**Exercise:** Open `extensions/minimal.ts`. Annotate every TypeScript-specific construct
with a comment explaining its C/Java equivalent. The goal is to be able to read TS
syntax as fluently as you read Java.

---

### 0.4 Modern JavaScript Tooling

**Theory: The toolchain**

| Tool | Role | C/Java Equivalent |
|------|------|-------------------|
| Bun | Runtime + package manager | JVM + Maven |
| npm/package.json | Dependency manifest | pom.xml / CMakeLists.txt |
| node_modules/ | Installed packages | .m2 repository / lib/ directory |
| jiti | JIT TypeScript compiler | On-the-fly javac (no build step) |
| just | Task runner | Makefile |

The key insight about jiti: Pi loads extension `.ts` files at runtime without a compile
step. jiti compiles TypeScript to JavaScript in memory on demand. This means you can
edit an extension file and re-run `just ext-minimal` with no build step.

**Files to read:**

- `package.json` (9 lines) — declares one dependency (`yaml`)
- `justfile` (107 lines) — all runnable recipes, see `set dotenv-load := true`
- `.env.sample` (20 lines) — required API key variables
- `.pi/settings.json` (4 lines) — workspace theme and prompts config

**Exercise:** Run `bun install` from the project root. Open `node_modules/` and find the
`yaml` package. Understand why it is there (damage-control.ts parses YAML rules). Now
run `just` with no arguments and read the recipe list. Map each recipe to its extension
file in `extensions/`.

**Self-assessment checkpoint 0:** Can you explain in one sentence what each of the
following does: LLM, context window, tool call, jiti, justfile dotenv-load? Can you
read `extensions/minimal.ts` from top to bottom without getting stuck?

---

## Phase 1: Hello World — Your First Extension

**Goal:** Run existing extensions, understand the universal extension pattern, write your
first modification.

---

### 1.1 Run Plain Pi

**Command:** `just pi`

Observe the TUI layout. From top to bottom:

```
HEADER       — Pi version, session ID, model name
WIDGET AREA  — empty by default; extensions put content here
RESPONSE     — conversation history, tool call entries
INPUT EDITOR — where you type; Enter submits, Shift+Enter = newline
FOOTER       — default: model name + context usage %
STATUS LINE  — one-line status messages
```

Ask Pi to write a function that reverses a string. Watch the tool calls appear in the
response area. Note the context percentage in the footer increase slightly.

**Exercise:** Ask Pi: "List the TypeScript files in the extensions/ directory and tell
me one sentence about each." Watch which tools fire. Count the tool calls. Check your
context percentage before and after.

---

### 1.2 Study pure-focus.ts — The Minimum Viable Extension

**File:** `extensions/pure-focus.ts` (25 lines)

**Read the entire file now before continuing.**

The universal extension pattern:

```typescript
// 1. Import the ExtensionAPI type
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

// 2. Export a default function that receives `pi` (the extension API)
export default function (pi: ExtensionAPI) {

    // 3. Register event handlers at the TOP LEVEL (not inside other handlers)
    pi.on("session_start", async (_event, ctx) => {
        // 4. Use ctx (ExtensionContext) to interact with the UI and session
        ctx.ui.setFooter((_tui, _theme, _footerData) => ({
            dispose: () => {},
            invalidate() {},
            render(_width: number): string[] {
                return [];  // empty array = no footer lines rendered
            },
        }));
    });
}
```

**Theory: The extension lifecycle**

1. Pi starts with `-e extensions/pure-focus.ts`
2. jiti loads the file and calls the default exported function with the `pi` API object
3. The function registers event handlers by calling `pi.on(eventName, handler)`
4. When the session starts, `session_start` fires and the handler runs
5. The handler calls `ctx.ui.setFooter(...)` to replace the default footer

The footer renderer is a function that returns an object with a `render(width)` method.
`render` returns an array of strings — one string per line. Returning an empty array
suppresses the footer entirely. This is the pure-focus trick.

**Analogy for C developers:** The extension is a plugin that registers callbacks. Pi is
the host process. The event system is a signal/slot mechanism (like Qt). The footer
renderer is a vtable with a `render` virtual method.

**Run it:** `just ext-pure-focus`

**Exercise:** Copy `extensions/pure-focus.ts` to `extensions/my-first.ts`. Modify the
`render` function to return `[" Hello from my extension! "]` instead of `[]`. Run it
with `pi -e extensions/my-first.ts`. Confirm you see the text in the footer.

---

### 1.3 Study minimal.ts — Footer with Live Data

**File:** `extensions/minimal.ts` (34 lines)

**Read the entire file now.**

New concepts introduced:

```typescript
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";
```

These two functions solve a real problem: terminal strings contain invisible ANSI escape
codes for colors. `"hello"` is 5 bytes wide. `"\x1b[32mhello\x1b[0m"` (green "hello")
is 16 bytes wide but still displays as 5 characters. `visibleWidth` returns the visual
width (5), not the byte width (16). `truncateToWidth` cuts a string at a specific visual
width, ignoring ANSI escapes.

```typescript
const model = ctx.model?.id || "no-model";       // optional chaining
const usage = ctx.getContextUsage();              // live data from Pi
const pct = (usage && usage.percent !== null) ? usage.percent : 0;
const filled = Math.round(pct / 10);             // 0–10 blocks
const bar = "#".repeat(filled) + "-".repeat(10 - filled);

const left = theme.fg("dim", ` ${model}`);        // themed color
const right = theme.fg("dim", `[${bar}] ${Math.round(pct)}% `);
const pad = " ".repeat(Math.max(1, width - visibleWidth(left) - visibleWidth(right)));
return [truncateToWidth(left + pad + right, width)];
```

This pattern — left content + padding + right content — is used in almost every footer
in this codebase. Memorize it.

**Theme tokens:** The five semantic color names are `success`, `accent`, `warning`,
`dim`, and `muted`. Read `THEME.md` (30 lines) to understand what each one means
visually and when to use which.

**Run it:** `just ext-minimal`

**Exercise:** Modify the bar in `extensions/minimal.ts` to use `█` (Unicode block) for
filled and `░` (Unicode light block) for empty. Note that `visibleWidth` must still
work — each Unicode block is one character wide, so the math is the same.

---

### 1.4 Study themeMap.ts — Shared Infrastructure

**File:** `extensions/themeMap.ts` (144 lines)

**Read the entire file now.**

This file does two things:

1. Defines `THEME_MAP` — a mapping from extension name to theme name. When you load
   `minimal.ts`, it calls `applyExtensionDefaults(import.meta.url, ctx)`, which looks
   up `"minimal"` in the map and loads the `"synthwave"` theme.

2. Implements `primaryExtensionName()` — reads `process.argv` to find the first `-e`
   flag. When extensions are stacked (`pi -e minimal.ts -e theme-cycler.ts`), both
   extensions call `applyExtensionTheme` during `session_start`. Without coordination,
   the last one to run would win. `primaryExtensionName` lets all stacked extensions
   agree on "the first one listed is the theme authority" — so only the primary extension
   applies its theme.

**Theory: Extension stacking**

Multiple `-e` flags load multiple extensions. Each one calls its `session_start` handler
independently. State is *not* shared between extensions by default (each file has its own
closure). The only shared coordination mechanism shown here is reading `process.argv` to
derive a deterministic answer without any shared mutable state.

**Exercise:** Add an entry to `THEME_MAP` for your `my-first.ts` extension:
`"my-first": "dracula"`. Run `pi -e extensions/my-first.ts` and confirm the dracula
theme loads. Then stack it with another extension: `pi -e extensions/my-first.ts
-e extensions/minimal.ts`. Verify your theme is used (not minimal's synthwave), because
your extension is listed first.

**Self-assessment checkpoint 1:** Can you explain what `ctx.ui.setFooter` does? Can you
explain what `applyExtensionDefaults` does? Can you create an extension that shows
"Hello [model-name]" in the footer in the correct theme color?

---

## Phase 2: Events and Interception

**Goal:** Understand the event system deeply. Learn to block, modify, and react to the
three most important event types: input, tool_call, and before_agent_start.

---

### 2.1 Study purpose-gate.ts — Input Interception

**File:** `extensions/purpose-gate.ts` (85 lines)

**Read the entire file now.**

Three new events are used here:

**`session_start`** — same as before, used to kick off the async purpose dialog

**`input`** — fires when the user submits a message. The handler can return:
- `{ action: "continue" }` — let the message proceed to the AI
- `{ action: "handled" }` — consume the input; the AI never sees it

**`before_agent_start`** — fires just before the AI processes each prompt. The handler
can return `{ systemPrompt: string }` to replace or augment the system prompt for this
turn. Returning nothing leaves it unchanged.

**Theory: Blocking with `action: "handled"`**

```typescript
pi.on("input", async (_event, ctx) => {
    if (!purpose) {
        ctx.ui.notify("Set a purpose first.", "warning");
        return { action: "handled" as const };  // blocks the message
    }
    return { action: "continue" as const };
});
```

The `as const` is a TypeScript assertion that narrows the type from `string` to the
literal `"handled"`. Without it, TypeScript infers `string`, which does not match the
expected union type. This pattern appears throughout the codebase.

**`ctx.ui.input(title, placeholder)`** — shows a text input dialog. Returns a `Promise<string | undefined>`. If the user cancels, returns `undefined`. The `void askForPurpose(ctx)` call means "start this async function but do not await it" — the session_start handler returns immediately while the dialog runs concurrently.

**The widget pattern:**

```typescript
ctx.ui.setWidget("purpose", () => {
    return {
        render(width: number): string[] { ... },
        invalidate() {},
    };
});
```

A widget is a persistent display area above the input editor. The key `"purpose"` is the
widget's identity — calling `setWidget` with the same key replaces the previous widget.
Calling `setWidget("purpose", undefined)` removes it.

**Exercise:** Create an extension `extensions/confirm-gate.ts` that:
1. On `session_start`, does nothing special
2. On `input`, intercepts every message and shows a `ctx.ui.confirm()` dialog asking
   "Send this message?" — allows the message through only if confirmed, blocks it
   otherwise. Read the API: `ctx.ui.confirm(title, body, options?)` returns `Promise<boolean>`.

---

### 2.2 Study damage-control.ts — Tool Call Interception

**File:** `extensions/damage-control.ts` (206 lines)
**Also read:** `.pi/damage-control-rules.yaml` (first 80 lines to understand structure)

**Read both files now.**

**`tool_call`** — fires before any tool executes. The handler can return:
- `{ block: false }` — allow the tool
- `{ block: true, reason: string }` — block the tool; `reason` is shown to the AI

**`isToolCallEventType(toolName, event)`** — a type-narrowing helper. The `event` passed
to `tool_call` is a union type — it could be a `bash` call, a `read` call, a `write`
call, etc. Each one has different `event.input` fields. Using `isToolCallEventType` both
checks the tool name AND narrows the TypeScript type, so `event.input.command` is only
valid inside the bash branch:

```typescript
if (isToolCallEventType("bash", event)) {
    const command = event.input.command;  // TypeScript knows this exists
}
if (isToolCallEventType("read", event)) {
    const path = event.input.path;        // TypeScript knows THIS exists
}
```

**Theory: The `ask` pattern**

Some rules in damage-control-rules.yaml have `ask: true`. For these, the extension shows
a confirmation dialog instead of outright blocking. The `ctx.ui.confirm()` call blocks
the `tool_call` handler while the user decides. This is only possible because extension
handlers are async — the event system awaits the returned promise. A shell hook
equivalent would require polling or IPC; Pi's in-process model makes it trivial.

**`pi.appendEntry(key, data)`** — persists custom data to the session JSONL file under
the given key. This creates an audit log of all blocked tool calls that survives session
restarts. This is a unique Pi feature with no direct Claude Code equivalent.

**Exercise:** Add a new rule to `.pi/damage-control-rules.yaml`:

```yaml
- pattern: '\bcurl\b.*-X\s+DELETE'
  reason: curl DELETE request (use safe HTTP methods for reading)
```

Test it by asking Pi: "Run `curl -X DELETE http://example.com/api/users/1`". Confirm
it is blocked.

---

### 2.3 Study system-select.ts — System Prompt Modification

**File:** `extensions/system-select.ts` (168 lines)

**Read the entire file now.**

New concept: `pi.registerCommand(name, { description, handler })`

This registers a slash command. Typing `/system` in the input editor calls the `handler`
function. The handler receives `(args: string, ctx: ExtensionContext)` where `args` is
everything after `/system` (e.g., for `/system planner`, args would be `"planner"`).

**Theory: Frontmatter parsing**

Agent definition files use YAML frontmatter:

```markdown
---
name: planner
description: Architecture and implementation planning
tools: read,grep,find,ls
---
You are a planner agent. Analyze requirements...
```

The `parseFrontmatter` function splits on the `---` delimiters, parses the YAML header,
and returns the body (the system prompt text). This is a recurring pattern in this
codebase — the same approach appears in cross-agent.ts, agent-team.ts, agent-chain.ts.

**`pi.getActiveTools()` and `pi.setActiveTools(tools)`** — read and write the list of
tool names the AI is allowed to use. Setting tools to `["read", "grep"]` means the AI
cannot call `bash` or `write` even if it tries — Pi will reject the tool call.

**`ctx.ui.select(title, items)`** — shows a selection list dialog. Returns the selected
string or `undefined` if cancelled. Items are displayed in order.

**Exercise:** Create a new agent file `.pi/agents/explainer.md`:

```markdown
---
name: explainer
description: Explains code in plain language
tools: read,grep,ls
---
You are an explainer agent. When asked about code, read the relevant files
and explain them in plain language suitable for a junior developer.
Never write or modify any files.
```

Run `just ext-system-select`. Type `/system` and select your new explainer agent.
Ask Pi to explain how `extensions/minimal.ts` works. Verify the explanation is plain
and clear and that Pi does not try to write any files.

**Self-assessment checkpoint 2:** Can you explain the difference between `input`,
`tool_call`, and `before_agent_start` events? Can you write a handler for each that
demonstrates its distinctive behavior?

---

## Phase 3: Rich UI — Footers, Widgets, and Overlays

**Goal:** Learn to build every type of custom UI surface Pi offers.

---

### 3.1 Study tool-counter.ts — Rich Multi-Line Footer

**File:** `extensions/tool-counter.ts` (103 lines)

**Read the entire file now.**

New concepts:

**`tool_execution_end`** — fires after a tool finishes executing (contrast with
`tool_call`, which fires before). Used here to increment per-tool counters.

**Session branch traversal:**

```typescript
for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.role === "assistant") {
        const m = entry.message as AssistantMessage;
        tokIn += m.usage.input;
        tokOut += m.usage.output;
        cost += m.usage.cost.total;
    }
}
```

`getBranch()` returns the current conversation as a flat array of session entries. An
`AssistantMessage` has a `usage` field with token counts and cost. This is how the
footer accumulates totals across the whole session — it iterates all past messages every
time the footer re-renders.

**`footerData.getGitBranch()` and `footerData.onBranchChange(callback)`** — utilities
provided by Pi for git-aware footers. `onBranchChange` returns an unsubscribe function,
which is stored in `dispose`. When Pi tears down the footer, it calls `dispose()` to
prevent memory leaks.

**Multi-line footer:** `render(width)` can return more than one string. Each string is
one line. This footer returns two lines.

**Exercise:** Add a third footer line to tool-counter.ts that shows the total number of
assistant messages in the current session (count entries where `entry.message.role ===
"assistant"`). Format it like: `  turns: 4  `.

---

### 3.2 Study tool-counter-widget.ts — Widget with Components

**File:** `extensions/tool-counter-widget.ts` (69 lines)

**Read the entire file now.**

**`ctx.ui.setWidget(key, factory, options?)`** — registers a widget. The factory
function receives `(tui, theme)` and returns an object with `render` and `invalidate`.

**TUI component imports:**

```typescript
import { Box, Text } from "@mariozechner/pi-tui";
```

Pi provides a small set of composable components:
- `Text(content, marginTop, marginBottom)` — wraps a text string with margins
- `Box(marginTop, marginBottom, decorator?)` — a container with optional line decoration
- `Container` — a vertical stack of child components
- `Spacer(height)` — empty space
- `Markdown(content, marginTop, marginBottom, mdTheme)` — rendered markdown

`text.setText(newContent)` + `text.invalidate()` triggers a re-render. `invalidate()`
marks the component as needing repaint on the next render cycle.

**`{ placement: "belowEditor" }` option** — by default widgets appear above the editor.
The theme-cycler's swatch widget uses `"belowEditor"` to appear below.

**ANSI background color:**

```typescript
function bg(rgb: number[], s: string): string {
    return `\x1b[48;2;${rgb[0]};${rgb[1]};${rgb[2]}m${s}\x1b[49m`;
}
```

`\x1b[48;2;R;G;Bm` sets background color (24-bit). `\x1b[49m` resets to default.
`\x1b[38;2;R;G;Bm` sets foreground color. `\x1b[0m` resets everything. These are
standard ANSI escape sequences — the same ones used in every colored terminal application.

**Exercise:** Create a widget extension `extensions/live-clock.ts` that:
1. Registers a widget using `ctx.ui.setWidget`
2. Shows the current time formatted as `HH:MM:SS`
3. Updates every second using `setInterval`
4. Invalidates the widget on each tick so it re-renders

Hint: store the `tui` reference from the factory function. Call
`tui.requestRender()` in the interval callback to force a repaint.

---

### 3.3 Study session-replay.ts — Full Overlay

**File:** `extensions/session-replay.ts` (217 lines)

**Read the entire file now.**

**`ctx.ui.custom(factory, options)`** — shows a full-screen custom overlay. The factory
returns `{ render, handleInput, invalidate }`. The `done` callback closes the overlay.
Options include `{ overlay: true, overlayOptions: { width: "80%", anchor: "center" } }`.

**Keyboard navigation:**

```typescript
import { matchesKey, Key } from "@mariozechner/pi-tui";

handleInput(data: string, tui: any): void {
    if (matchesKey(data, Key.up)) { ... }
    if (matchesKey(data, Key.enter)) { ... }
    if (matchesKey(data, Key.escape)) { this.onDone(); }
    tui.requestRender();
}
```

`data` is a raw string of keyboard input. `Key.up`, `Key.down`, `Key.enter`,
`Key.escape` are constants. `matchesKey` handles special key sequences (arrow keys send
multi-byte escape sequences like `\x1b[A`).

**`SessionReplayUI` class pattern:** Note that this is the only file in the codebase that
defines a class. It encapsulates scroll state (`selectedIndex`, `scrollOffset`,
`expandedIndex`) and rendering logic. The custom overlay passes keyboard events to
`handleInput` and rendering calls to `render`. This is an MVC-like pattern — the class
is both model and view.

**`DynamicBorder`** — a Pi-specific component that renders a horizontal line the full
width of the terminal using `─` characters, accepting a colorize function.

**Exercise:** Study the `ensureVisible` method in `SessionReplayUI`. Draw a diagram of
the scroll window logic. Trace what happens to `scrollOffset` when you press Down until
`selectedIndex` would go off screen. Confirm your understanding is correct by running
`just ext-session-replay` and navigating with arrow keys.

---

### 3.4 Study theme-cycler.ts — Shortcuts, Commands, and Timers

**File:** `extensions/theme-cycler.ts` (182 lines)

**Read the entire file now.**

**`pi.registerShortcut(key, { description, handler })`** — registers a keyboard shortcut.
The key string uses the format `"ctrl+x"`. Reserved keys (see `RESERVED_KEYS.md`) are
silently ignored. Safe keys for extensions: `ctrl+x`, `ctrl+q`.

**`ctx.ui.getAllThemes()`** — returns the full list of available themes. Each entry has
`{ name, path }`. `ctx.ui.theme.name` is the currently active theme name.

**`ctx.ui.setTheme(name)`** — returns `{ success: boolean, error?: string }`. Always
check the return value.

**`ctx.ui.setStatus(key, text)`** — sets a named status line entry. Multiple extensions
can each own a named status slot. The status line aggregates all active slots.

**Timer cleanup pattern:**

```typescript
let swatchTimer: ReturnType<typeof setTimeout> | null = null;

// On shortcut:
swatchTimer = setTimeout(() => {
    ctx.ui.setWidget("theme-swatch", undefined);
    swatchTimer = null;
}, 3000);

// On session shutdown:
pi.on("session_shutdown", async () => {
    if (swatchTimer) {
        clearTimeout(swatchTimer);
        swatchTimer = null;
    }
});
```

Always clean up timers in `session_shutdown`. A leaked timer will attempt to call
`ctx.ui.setWidget` after the session ends, causing errors.

**Exercise:** Add a shortcut `ctrl+h` (with a warning that it may alias backspace in some
terminals — see RESERVED_KEYS.md) that shows a "Help" overlay listing all registered
shortcuts. Use `ctx.ui.custom` to display the overlay.

**Self-assessment checkpoint 3:** Can you create an extension with a footer showing a
live clock AND a below-editor widget showing total word count of AI responses in the
current session?

---

## Phase 4: Tools — The Agent's Hands

**Goal:** Learn to register custom tools, design TypeBox schemas, handle streaming
results, and reconstruct state from session history.

---

### 4.1 Study tilldone.ts — Full Tool Registration

**File:** `extensions/tilldone.ts` (727 lines)

This is the most instructive single file in the repository. Read it in sections over
multiple sessions. Do not rush it.

**Section 1 (lines 1–62): Types and schema**

```typescript
import { StringEnum } from "@mariozechner/pi-ai";
import { Type } from "@sinclair/typebox";

const TillDoneParams = Type.Object({
    action: StringEnum(["new-list", "add", "toggle", "remove", "update", "list", "clear"] as const),
    text: Type.Optional(Type.String({ description: "Task text..." })),
    id: Type.Optional(Type.Number({ description: "Task ID..." })),
});
```

TypeBox generates JSON Schema at runtime. The schema is passed to the LLM so it knows
what parameters the tool accepts. `StringEnum` creates a union of string literals —
the LLM must pass exactly one of those strings. `Type.Optional` makes a field optional.
The `description` field is critical — the LLM reads it to understand how to use the
parameter.

**Section 2: `pi.registerTool()` — the full API**

```typescript
pi.registerTool({
    name: "tilldone",
    label: "TillDone",
    description: "Manage the task list. MUST use 'new-list' + 'add' before any other tools.",
    parameters: TillDoneParams,

    async execute(toolCallId, params, signal, onUpdate, ctx) {
        // toolCallId: unique ID for this call (used to match with result later)
        // params: the typed parameters — TypeScript infers the type from TillDoneParams
        // signal: AbortSignal — check signal.aborted to support cancellation
        // onUpdate(text): stream partial results to the UI while executing
        // ctx: ExtensionContext

        // Return: { content: string, details?: any }
        // content = what the LLM sees in its context
        // details = what gets persisted to session (survives restart)
        return { content: "Done", details: { action, tasks, nextId } };
    },

    renderCall(args, theme) {
        // Returns a string[] rendered in the response area when the tool is CALLED
        // args = the raw parameters object
        return [`  [TillDone] ${args.action}`];
    },

    renderResult(result, options, theme) {
        // result.details = the `details` object from execute()
        // Returns a TUI component (not string[])
        // Used to reconstruct state from previous sessions
        return new Text(...);
    },
});
```

**Theory: State reconstruction from session**

When Pi resumes a session, it replays all session entries. For each past tool call, Pi
calls `renderResult(result, ...)` again. Tilldone uses this to rebuild its task list
from the session history — it scans the session for all previous tilldone results and
replays the actions to reconstruct the current state. This is the CQRS pattern (Command
Query Responsibility Segregation) applied to session management.

```typescript
// Scan session for previous tilldone results (state reconstruction)
for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "tool_result" && entry.toolName === "tilldone") {
        const d = entry.details as TillDoneDetails;
        // replay actions to rebuild state...
    }
}
```

**Theory: Blocking gate via `tool_call`**

Tilldone blocks ALL other tools until at least one task has been added:

```typescript
pi.on("tool_call", async (event) => {
    if (event.toolName === "tilldone") return { block: false };
    if (tasks.length === 0) {
        return { block: true, reason: "Use tilldone to define tasks first." };
    }
    return { block: false };
});
```

**Theory: Auto-nudge via `agent_end`**

When the agent stops responding, tilldone checks if any tasks are incomplete and if so
sends a follow-up message:

```typescript
pi.on("agent_end", async (_event, ctx) => {
    const incomplete = tasks.filter(t => t.status !== "done");
    if (incomplete.length > 0) {
        await pi.sendUserMessage("You have incomplete tasks. Continue or mark them done.");
    }
});
```

**Exercise:** Create a `"notes"` tool that lets the agent:
1. Save a named note: `{ action: "save", name: string, content: string }`
2. Retrieve a note by name: `{ action: "get", name: string }`
3. List all notes: `{ action: "list" }`

Store notes in a `Map<string, string>`. Implement state reconstruction from session
history. Add a `/notes` command that shows all current notes.

---

### 4.2 Study cross-agent.ts — Command Registration at Scale

**File:** `extensions/cross-agent.ts` (265 lines)

**Read the entire file now.**

**Theory: Dynamic command registration**

`pi.registerCommand` can be called at any time, including inside `session_start`. This
allows commands to be discovered dynamically from the filesystem at boot:

```typescript
for (const cmd of commands) {
    pi.registerCommand(cmd.name, {
        description: `[${source}] ${cmd.description}`,
        handler: async (args) => {
            pi.sendUserMessage(expandArgs(cmd.content, args || ""));
        },
    });
}
```

**`expandArgs` template substitution:**

```typescript
function expandArgs(template: string, args: string): string {
    const parts = args.split(/\s+/).filter(Boolean);
    let result = template;
    result = result.replace(/\$ARGUMENTS|\$@/g, args);   // all args
    for (let i = 0; i < parts.length; i++) {
        result = result.replaceAll(`$${i + 1}`, parts[i]); // $1, $2, ...
    }
    return result;
}
```

This is how `.claude/commands/*.md` prompt templates work: `$1` is the first argument,
`$ARGUMENTS` is all arguments joined. The same convention works in Pi, Claude Code,
Gemini CLI, and Codex — they share the agent skills standard.

**`pi.sendUserMessage(text)`** — sends a message to the AI as if the user typed it.
Used here to inject a command's prompt content into the conversation.

**Exercise:** Create a `.claude/commands/explain.md` file:

```markdown
---
description: Explain a file or concept
---
Please explain the following in clear, simple terms suitable for a junior developer: $ARGUMENTS
```

Run `just ext-cross-agent`. Type `/explain extensions/minimal.ts`. Confirm the command
is discovered and the prompt is sent to the AI with the file path substituted.

**Self-assessment checkpoint 4:** Can you register a tool with a TypeBox schema, execute
it, return structured results, and reconstruct its state from session history?

---

## Phase 5: Multi-Agent Orchestration

**Goal:** Understand and implement all three multi-agent patterns: background worker,
pipeline, and dispatcher.

---

### 5.1 Theory: Multi-Agent Patterns

Before reading any code, understand the three architectural patterns:

**Background Worker Pattern (subagent-widget)**
- Main agent continues working while a separate process handles a long task
- Communication: JSONL event stream from subprocess stdout
- State: each worker has its own session file for continuability
- Use case: "analyze this large codebase while I continue asking questions"

**Pipeline Pattern (agent-chain)**
- A fixed sequence: output of step N becomes input to step N+1
- Each step uses a different agent (different system prompt, different tools)
- Use case: "plan then build then review" where each phase needs different constraints

**Dispatcher Pattern (agent-team)**
- Primary agent has NO codebase tools — it can ONLY call `dispatch_agent`
- Primary reads the user's request and selects the right specialist
- Specialist agents run in subprocesses with their own tool sets
- Use case: "route complex requests to the right specialist automatically"

**Parallel Research Pattern (pi-pi)**
- All experts are queried simultaneously via `Promise.allSettled`
- Each expert is domain-specific (TUI, themes, extensions, config...)
- Results are aggregated before the primary agent synthesizes
- Use case: "gather all relevant documentation before writing anything"

**Theory: How subprocess Pi agents work**

```
Main Pi process
  → spawns: pi --mode json -p "task" --session-file path.jsonl
  → reads: stdout line-by-line (JSONL events)
  → each line is a JSON object with a `type` field

Event types to watch:
  { type: "message_update", delta: { type: "text_delta", text: "..." } }
  { type: "tool_execution_start", toolName: "read", ... }
  { type: "agent_end", ... }
```

`--mode json` makes Pi emit JSONL instead of rendering a TUI. `-p "task"` submits a
prompt non-interactively. `--session-file path.jsonl` saves the session for `/subcont`
resumption.

---

### 5.2 Study subagent-widget.ts — Background Workers

**File:** `extensions/subagent-widget.ts` (482 lines)

Read in three passes:

**Pass 1 (lines 1–100): Types and widget rendering**

The `SubState` interface holds everything needed to render one agent widget: status,
task text, elapsed time, tool count, last output line. The `updateWidgets()` function
rebuilds all active widget renderers from current state.

**Pass 2 (lines 100–300): The /sub command and subprocess spawning**

```typescript
const proc = spawn("pi", [
    "--mode", "json",
    "--session-file", state.sessionFile,
    "-p", task,
], { cwd: ctx.cwd, env: process.env });

proc.stdout.on("data", (data: Buffer) => {
    for (const line of data.toString().split("\n")) {
        if (!line.trim()) continue;
        try {
            const event = JSON.parse(line);
            // handle event.type === "message_update", "tool_execution_start", etc.
        } catch {}
    }
});
```

The subprocess writes one JSON object per line to stdout. The parent process reads
chunks from stdout, splits on newlines, and parses each line. Incomplete lines from
buffer boundaries are handled by accumulating partial lines.

**Pass 3 (lines 300–482): /subcont and /subrm commands**

`/subcont id prompt` — resumes a finished agent by spawning a new subprocess using the
same session file. The session file contains all previous messages, so the agent
remembers its prior work.

**Exercise:** Run `just ext-subagent-widget`. Type `/sub read extensions/minimal.ts and
summarize it in 3 bullet points`. Watch the widget appear and update in real time.
After it finishes, type `/subcont 1 now extend your summary to include the theme color
tokens it uses`. Watch the agent resume with its prior context.

---

### 5.3 Study agent-team.ts — Dispatcher Pattern

**File:** `extensions/agent-team.ts` (734 lines)

Read in four passes:

**Pass 1 (lines 1–100): Types and agent loading**

`AgentDef` holds the agent's name, description, tools string, and system prompt.
`AgentState` adds runtime state: status, elapsed time, run count, session file.

Agent files are loaded from multiple directories at boot. The `teams.yaml` file maps
team names to agent name lists. The extension scans all agent files and builds the
full catalog.

**Pass 2 (lines 100–300): The dispatch_agent tool**

The primary agent has only one tool: `dispatch_agent`. Its schema requires selecting
an agent name (from the team), providing a task description, and optionally a session
continuation flag.

When `dispatch_agent` is called:
1. Look up the named agent in the catalog
2. Spawn `pi --mode json --system-prompt <agent-system-prompt> --tools <agent-tools> -p <task>`
3. Stream the subprocess output to update the agent's widget card
4. Return the final output as the tool result

**Pass 3 (lines 300–500): Grid dashboard rendering**

The grid dashboard is a widget that shows all team members as cards. Each card shows:
- Agent name and status icon
- Elapsed time if running
- Context bar if known
- Last line of output (truncated)

The grid reflows into N columns based on terminal width and the configured column count.

**Pass 4 (lines 500–734): Team management commands**

`/agents-team` — shows a selection dialog listing all teams from `teams.yaml`. Selecting
a team replaces the current team and rebuilds the grid widget.

**Theory: No-codebase-tools constraint**

```typescript
pi.on("session_start", async (_event, ctx) => {
    pi.setActiveTools(["dispatch_agent"]);
    // ...
});
```

Restricting the primary agent to only `dispatch_agent` forces it to delegate everything.
This is the principle of least privilege — the dispatcher should coordinate, not
implement.

**Exercise:** Add a new agent file `.pi/agents/tester.md`:

```markdown
---
name: tester
description: Writes and runs tests for implemented code
tools: read,write,bash,grep
---
You are a tester agent. Write unit tests for the code provided.
Run them with bash. Report which tests pass and which fail.
```

Add `tester` to the `plan-build` team in `.pi/agents/teams.yaml`. Run `just ext-agent-team`, select the `plan-build` team, and ask Pi to implement and test a simple function.

---

### 5.4 Study agent-chain.ts — Pipeline Pattern

**File:** `extensions/agent-chain.ts` (797 lines)

**Theory: The chain execution model**

```yaml
# .pi/agents/agent-chain.yaml
plan-build-review:
  steps:
    - agent: planner
      prompt: "Plan the implementation for: $INPUT"
    - agent: builder
      prompt: "Implement the following plan:\n\n$INPUT"
    - agent: reviewer
      prompt: "Review this implementation:\n\n$INPUT"
```

`$INPUT` in step 1 = the user's original prompt.
`$INPUT` in step 2 = the output of step 1.
`$INPUT` in step N = the output of step N-1.
`$ORIGINAL` always = the user's original prompt regardless of step.

The chain is triggered by the `run_chain` tool. The primary agent calls
`run_chain({ chain: "plan-build-review", input: "user request" })`.

**Read the file in two passes:**

**Pass 1 (lines 1–200): Types, YAML loading, agent loading**

Compare `ChainDef`, `ChainStep`, `AgentDef` to their equivalents in agent-team.ts.
Note how the YAML parsing is nearly identical — this is intentional shared convention.

**Pass 2 (lines 200–797): run_chain tool and step execution**

The `execute` function runs steps sequentially. After each step, the result becomes
`$INPUT` for the next step's prompt template. Progress is shown in the footer.

**Exercise:** Add a new chain to `.pi/agents/agent-chain.yaml`:

```yaml
scout-build:
  description: "Explore then build — recon before implementation"
  steps:
    - agent: scout
      prompt: "Explore the codebase relevant to this request and report your findings: $INPUT"
    - agent: builder
      prompt: "Based on this reconnaissance report, implement the request.\n\nOriginal request: $ORIGINAL\n\nRecon report:\n$INPUT"
```

Run `just ext-agent-chain`, select the `scout-build` chain, and ask it to add error
handling to the minimal extension.

---

### 5.5 Study pi-pi.ts — Parallel Research Pattern

**File:** `extensions/pi-pi.ts` (633 lines)

Read in two passes:

**Pass 1 (lines 1–200): Expert definitions and the query_experts tool**

Expert agents live in `.pi/agents/pi-pi/`. Each one is a domain specialist: `ext-expert`
knows the extension API, `theme-expert` knows color tokens and theme structure,
`tui-expert` knows TUI components, and so on.

The `query_experts` tool accepts a list of `{ expert, question }` pairs. The primary
agent calls it once with multiple questions — one per relevant expert.

**Pass 2 (lines 200–633): Parallel execution with Promise.allSettled**

```typescript
const results = await Promise.allSettled(
    queries.map(async ({ expert, question }) => {
        const result = await runExpert(expert, question, ctx);
        return { expert, result };
    })
);
```

`Promise.allSettled` runs all queries in parallel and waits for all of them, collecting
both successes and failures. `Promise.all` would abort on the first failure —
`Promise.allSettled` is safer for research patterns where one failing expert should not
cancel the others.

Each expert gets a distinct color in the grid dashboard so you can track their progress
visually.

**Exercise:** Run `just ext-pi-pi`. Ask: "Build a simple extension that shows a word
count of all user messages in the footer." Watch the experts research in parallel. Review
the generated extension code — evaluate its quality against what you now know about
correct extension patterns.

**Self-assessment checkpoint 5:** Can you explain the trade-offs between background
worker, pipeline, and dispatcher patterns? For each pattern, give a real-world use case
where it is the correct choice and two where it would be the wrong choice.

---

## Phase 6: Configuration and Safety

**Goal:** Understand the full configuration stack and design principled safety rules.

---

### 6.1 The Configuration Layer

**Files to read:**

- `.pi/settings.json` (4 lines)
- `.env.sample` (20 lines)
- `justfile` (107 lines, full read)
- `package.json` (9 lines)

**Theory: Configuration precedence**

Pi resolves settings from multiple sources. Workspace settings in `.pi/settings.json`
apply to this project. User-level settings in `~/.pi/settings.json` apply globally.
Environment variables override both. The `justfile` uses `set dotenv-load := true` to
automatically source `.env` before every recipe — this is the correct way to inject
API keys without committing secrets.

**Understanding the justfile recipes:** Note how `ext-minimal` stacks two extensions:

```
ext-minimal:
    pi -e extensions/minimal.ts -e extensions/theme-cycler.ts
```

The `just all` recipe uses `just open` to open every extension in its own terminal
window. `just open` is a macOS-specific recipe (uses `osascript`) — on Linux/Windows,
run the `pi -e` commands directly.

**Exercise:** Modify `.pi/settings.json` to add `"model": "claude-sonnet-4-5"` as the
default model. Run `just pi` and verify the model is pre-selected. Then remove the
setting and verify the default is restored.

---

### 6.2 Deep Dive: Damage Control Rules

**File:** `.pi/damage-control-rules.yaml` (full file, ~280 lines)

**Read the entire file now.**

Categorize every rule by threat type:

| Category | Examples | Why it matters |
|----------|----------|----------------|
| Destructive deletion | `rm -rf`, `rmdir --recursive` | Unrecoverable data loss |
| Git history rewrite | `git reset --hard`, `git push --force` | Permanent commit loss |
| Credential exposure | `.env`, `~/.ssh/`, `*.pem`, `*.key` | Secret exfiltration |
| System file mutation | `/etc/`, `/usr/`, `/System/` | OS instability |
| Cloud destructive ops | `aws s3 rm --recursive`, `gcloud ... delete` | Production data loss |
| Database destructive | `DROP TABLE`, `DELETE` without `WHERE` | Data model destruction |

**Theory: Defense in depth**

No single safety measure is sufficient. The rules work as layers:
1. `bashToolPatterns` — catch dangerous shell commands by pattern
2. `zeroAccessPaths` — prevent reading credentials at all
3. `readOnlyPaths` — prevent modifying config files
4. `noDeletePaths` — prevent deleting critical project files

**Exercise:** Design a damage-control ruleset for a hypothetical production API server.
Your rules must cover:
1. Block all database DROP/TRUNCATE commands
2. Block reads of `/etc/ssl/` (server certs)
3. Allow reads but block writes to `config/production.yml`
4. Block deletion of `docker-compose.yml`
5. Ask-for-confirmation before `systemctl restart`

Write the YAML and add it to `.pi/damage-control-rules.yaml`. Test each rule.

---

### 6.3 Agent Definitions — Principle of Least Privilege

**Files to read** (all in `.pi/agents/`):

- `planner.md` — `tools: read,grep,find,ls` — read-only
- `builder.md` — `tools: read,write,edit,bash,grep,find,ls` — full access
- `scout.md` — `tools: read,grep,find,ls` — read-only
- `reviewer.md` — read to understand its tool set
- `documenter.md` — read to understand its tool set
- `red-team.md` — read to understand its tool set
- `teams.yaml` — see how agents combine into teams
- `agent-chain.yaml` — see how agents sequence into pipelines

**Theory: Why tool restriction is not just security**

Restricting a planner agent to read-only tools is not primarily a safety measure — it is
a correctness measure. A planner that can write files might start implementing before the
plan is approved. A reviewer that can edit files might "fix" things it was only supposed
to review. Tool restrictions encode the *contract* of each agent's role.

**Exercise:** Design a `security-auditor` agent:

```markdown
---
name: security-auditor
description: Audits code for security vulnerabilities
tools: read,grep,find,ls
---
You are a security auditor. Your job is to read code and identify
security vulnerabilities. You NEVER write or modify files. For each
vulnerability found, report: location (file:line), severity (low/medium/high/critical),
description, and recommended fix.
```

Add `security-auditor` to the `teams.yaml` under a new `security` team. Test it by
asking the team to audit `extensions/damage-control.ts` for security issues.

**Self-assessment checkpoint 6:** Can you design a complete agent team for a web
development project — with correct tool restrictions for each role — and explain why
each restriction exists?

---

## Phase 7: Integration and Advanced Patterns

**Goal:** Stack extensions, build complete systems from scratch, achieve mastery through
synthesis.

---

### 7.1 Extension Stacking Deep Dive

**Theory: Load order and event ordering**

When `pi -e A.ts -e B.ts -e C.ts` runs:
1. All three extension functions are called immediately in order (A, B, C)
2. All three register their `session_start` handlers
3. When `session_start` fires, handlers run in registration order (A first, then B, then C)
4. Theme: only the first extension (A) applies its theme (via `primaryExtensionName()`)
5. Commands: if A and B both register `/foo`, A's registration wins (first-wins)

**The `justfile` is the canonical source for intended stacking:**

- `ext-minimal` stacks `minimal + theme-cycler` (footer + theme shortcuts)
- `ext-subagent-widget` stacks `subagent-widget + pure-focus + theme-cycler`
- `ext-damage-control` stacks `damage-control + minimal + theme-cycler`

The pattern is: feature extension first, UI helpers after.

**Exercise:** Create a combination of three extensions that work together without
conflicts:

1. `extensions/minimal.ts` — footer
2. `extensions/tool-counter-widget.ts` — widget
3. `extensions/theme-cycler.ts` — shortcuts

Run `pi -e extensions/minimal.ts -e extensions/tool-counter-widget.ts
-e extensions/theme-cycler.ts`. Verify all three features work simultaneously. Check
the `RESERVED_KEYS.md` to confirm no key conflicts exist between theme-cycler's
shortcuts and any other extension.

---

### 7.2 Build a Complete Extension From Scratch

Design and implement a `session-stats` extension:

**Requirements:**
1. **Footer:** shows total input tokens, total output tokens, total cost, and elapsed
   session time (MM:SS format)
2. **Widget:** shows a per-tool breakdown table — one row per tool with call count and
   average execution time (if timing data is available)
3. **Command `/stats`:** opens a `ctx.ui.custom` overlay showing full session stats
   including per-model breakdown if the model was switched mid-session
4. **Shortcut `ctrl+x`:** toggles the widget visible/hidden

**Implementation order:**
1. Start with the footer (builds on tool-counter.ts patterns)
2. Add the widget (builds on tool-counter-widget.ts patterns)
3. Add the command (builds on session-replay.ts patterns)
4. Add the shortcut (builds on theme-cycler.ts patterns)

**Verify against these criteria:**
- Follows the `applyExtensionDefaults(import.meta.url, ctx)` convention
- Registers all event handlers at the top level (not inside `session_start`)
- Cleans up all timers and subscriptions in `session_shutdown`
- Uses semantic theme colors (`success`, `accent`, `warning`, `dim`, `muted`)
- Handles the case where no tools have run yet

---

### 7.3 Build a Mini Multi-Agent Workflow

Design and implement a minimal agent team:

**Agents to create:**
1. `.pi/agents/researcher.md` — `tools: read,grep,find,ls` — finds relevant code
2. `.pi/agents/writer.md` — `tools: read,write,edit` — writes prose documentation
3. `.pi/agents/editor.md` — `tools: read,write,edit` — reviews and improves prose

**Team configuration:**
Add a `docs-team: [researcher, writer, editor]` entry to `.pi/agents/teams.yaml`.

**Chain configuration:**
Add a `research-write-edit` chain to `.pi/agents/agent-chain.yaml`:
- Step 1 (researcher): "Find all relevant code and gather context for: $INPUT"
- Step 2 (writer): "Write clear documentation based on this research: $INPUT"
- Step 3 (editor): "Edit and improve this documentation for clarity and accuracy: $INPUT"

**Test both patterns:**
1. Run `just ext-agent-team`, select `docs-team`, ask it to document `extensions/minimal.ts`
2. Run `just ext-agent-chain`, select `research-write-edit`, ask it to document `extensions/minimal.ts`
3. Compare the outputs — which pattern produced better documentation, and why?

---

### 7.4 Code Quality Review

Review all extensions you have written against these criteria:

**Structural checks:**
- [ ] All `pi.on()` and `pi.registerTool()` calls are at the top level of the extension function
- [ ] No `pi.on()` calls inside other `pi.on()` callbacks
- [ ] All event handlers are `async` and return the correct type
- [ ] `import type` used for type-only imports

**Resource management:**
- [ ] Every `onBranchChange()` subscription has a corresponding `dispose()` call
- [ ] Every `setInterval()` has a corresponding `clearInterval()` in `session_shutdown`
- [ ] Every `setTimeout()` that may outlive the session is cleared in `session_shutdown`

**Theme and UI:**
- [ ] Colors use semantic tokens (`success`, `accent`, `warning`, `dim`, `muted`)
- [ ] All footer/widget render functions handle `width` correctly with `visibleWidth` and `truncateToWidth`
- [ ] ANSI reset codes are always closed (no unclosed color sequences)

**Self-assessment checkpoint 7:** Can you explain every line of every extension you have
written to someone else? Can you identify potential bugs in a new extension you have
never seen before?

---

## Phase 8: Theory Consolidation — The Big Picture

**Goal:** Step back from implementation and build a complete mental model of the system.

---

### 8.1 Architecture Mental Model

**The four packages:**

| Package | Role |
|---------|------|
| `@mariozechner/pi-ai` | LLM abstraction — talks to 324 models via 20+ providers |
| `@mariozechner/pi-coding-agent` | Agent loop — session management, tool execution, event system |
| `@mariozechner/pi-tui` | Terminal UI — rendering engine, components, themes |
| `@mariozechner/pi-coding-agent` (CLI) | Extension loading, command registration, the `pi` binary |

Extensions import from all four. The ExtensionAPI is the facade that gives extensions
controlled access to the agent loop and TUI without exposing internal state directly.

**Draw from memory:** Sketch the architecture showing:
- User input flowing from terminal to event system
- Event system notifying extensions
- Extension returning a block/continue decision
- LLM receiving the (possibly modified) prompt
- LLM tool call flowing through the `tool_call` event
- Tool result flowing back through `tool_result`
- Response streaming through `message_update` events
- Extensions rendering the footer/widget on each `requestRender()`

---

### 8.2 Design Patterns Catalog

Identify each pattern in the codebase:

| Pattern | Where | Why |
|---------|-------|-----|
| Event-driven (Observer) | `pi.on()` system | Decouples extensions from the agent loop |
| Plugin / Extension | Extension loading via `-e` | Allows opt-in behavior without modifying Pi |
| Dispatcher / Mediator | `agent-team.ts` | Primary agent coordinates specialists |
| Pipeline (Chain of Responsibility) | `agent-chain.ts` | Output of each stage feeds the next |
| Factory | `ctx.ui.setWidget(key, factory)` | Widget is recreated on each render cycle |
| State Machine | tilldone.ts task lifecycle | `idle → inprogress → done` with explicit transitions |
| CQRS (Command-Query Separation) | tilldone state reconstruction | Commands append to session; state rebuilt by replaying |
| Closure-based state | Every extension | Module-level `let` variables captured in closures |
| Structural typing | TypeBox schemas | Runtime-validated parameter schemas |

---

### 8.3 Trade-Off Analysis

For each decision in Pi's design, articulate the trade-off:

**TypeScript in-process vs shell hooks:**
- Pro: full session access, async, can block synchronously, can render UI
- Con: a crashing extension crashes Pi; extension must be written in TypeScript

**200-token system prompt vs 10,000-token prompt:**
- Pro: more tokens available for actual work; frontier models need less hand-holding
- Con: less guardrail guidance; agent must infer conventions from context

**YOLO default vs safe-by-default:**
- Pro: no friction; every operation works immediately
- Con: requires adding damage-control extension for any production-adjacent use

**No built-in subagents vs extension-based subprocess approach:**
- Pro: core stays minimal; each subagent strategy is independently optimizable
- Con: more code per use case; no built-in parallelism primitives

**Session JSONL tree vs linear conversation:**
- Pro: branching, forking, labeling, replay, state persistence across sessions
- Con: more complex than a simple array; requires understanding of session entry types

**`Promise.allSettled` for parallel research vs `Promise.all`:**
- Pro: one failing expert doesn't abort all others
- Con: must check `status === "fulfilled"` on every result; slightly more verbose

---

### 8.4 Final Project

Build a complete, production-quality extension ecosystem:

**Component 1: A `code-review-tool` extension**
- Registers a `request_review` tool that the AI calls with `{ file: string, changes: string }`
- The tool spawns a subprocess agent with the reviewer persona
- Streams the review result back via `onUpdate`
- Persists the review to session via `details`
- Reconstructs all previous reviews from session on startup

**Component 2: A `safety-plus` extension**
- Extends damage-control patterns with additional rules
- Adds a `ctx.ui.setWidget` showing a live "threat level" (none/low/medium/high)
- Logs all blocked operations to a local file (not just session)
- Reports a daily summary of blocked operations on `session_start`

**Component 3: Custom theme**
- Create `.pi/themes/my-theme.json` following the theme token structure
- Read an existing theme (e.g., `dracula.json`) to understand the schema
- Design a coherent color scheme with all 5 semantic tokens
- Add it to `THEME_MAP` in `themeMap.ts`

**Component 4: Documentation**
- Write a `README.md` for your extension ecosystem describing what each extension does,
  how to run them, and what events/tools each registers

**Delivery criteria:** All extensions follow project conventions from `CLAUDE.md`. All
event handlers are at the top level. All colors use semantic tokens. All resources are
cleaned up in `session_shutdown`. The stacked combination runs without errors.

---

## Appendix A: Recommended Reading Order

Start with the smallest files and build up to the largest. Never skip ahead.

| Order | File | Lines | Teaches |
|-------|------|-------|---------|
| 1 | `extensions/pure-focus.ts` | 25 | Absolute minimum: export, pi.on, setFooter |
| 2 | `extensions/minimal.ts` | 34 | Footer render, context usage, theme colors, visibleWidth |
| 3 | `extensions/themeMap.ts` | 144 | Shared infrastructure, primaryExtensionName, stacking |
| 4 | `extensions/purpose-gate.ts` | 85 | Input interception, before_agent_start, widgets, dialogs |
| 5 | `extensions/tool-counter-widget.ts` | 69 | setWidget, TUI components, tool_execution_end |
| 6 | `extensions/theme-cycler.ts` | 182 | registerShortcut, registerCommand, setStatus, timers |
| 7 | `extensions/tool-counter.ts` | 103 | Multi-line footer, session traversal, AssistantMessage |
| 8 | `extensions/cross-agent.ts` | 265 | Dynamic command registration, frontmatter, expandArgs |
| 9 | `extensions/system-select.ts` | 168 | System prompt manipulation, setActiveTools, select dialog |
| 10 | `extensions/session-replay.ts` | 217 | ctx.ui.custom, full overlay, Key/matchesKey, class UI |
| 11 | `extensions/damage-control.ts` | 206 | tool_call interception, isToolCallEventType, appendEntry |
| 12 | `extensions/tilldone.ts` | 727 | registerTool, TypeBox schemas, state reconstruction, CQRS |
| 13 | `extensions/subagent-widget.ts` | 482 | child_process.spawn, JSONL parsing, persistent sessions |
| 14 | `extensions/agent-chain.ts` | 797 | Pipeline pattern, YAML chain definitions, sequential steps |
| 15 | `extensions/agent-team.ts` | 734 | Dispatcher pattern, teams.yaml, grid dashboard, run_chain |
| 16 | `extensions/pi-pi.ts` | 633 | Parallel pattern, Promise.allSettled, expert subagents |

---

## Appendix B: Concept Dependency Graph

Read top-to-bottom. Each row depends on everything above it.

```
TypeScript syntax (const, async, interface, import/export)
  |
  v
Extension lifecycle (export default fn, pi.on, session_start, ctx)
  |
  v
Event system (input, tool_call, before_agent_start, tool_execution_end, agent_end)
  |
  +---> UI APIs (setFooter, setWidget, setStatus, notify, select, confirm, input, custom)
  |
  +---> Theme system (theme.fg(), semantic tokens, setTheme, getAllThemes)
  |
  v
Tool Registration (registerTool, TypeBox schema, execute, renderCall, renderResult)
  |
  v
Session Access (sessionManager.getBranch(), AssistantMessage, appendEntry)
  |
  v
Command & Shortcut Registration (registerCommand, registerShortcut, sendUserMessage)
  |
  v
Subprocess Orchestration (child_process.spawn, JSONL parsing, --mode json)
  |
  v
Multi-Agent Patterns (background worker, pipeline, dispatcher, parallel research)
  |
  v
Full Mastery (design, compose, debug, optimize complete extension ecosystems)
```

---

## Appendix C: Key Files Quick Reference

| File | Lines | Difficulty | What It Teaches |
|------|-------|------------|-----------------|
| `extensions/pure-focus.ts` | 25 | Beginner | Extension skeleton, setFooter(null) |
| `extensions/minimal.ts` | 34 | Beginner | Footer rendering, context usage, theme colors |
| `extensions/themeMap.ts` | 144 | Beginner | Shared infrastructure, extension stacking |
| `extensions/purpose-gate.ts` | 85 | Beginner | Input blocking, widget creation, before_agent_start |
| `extensions/tool-counter-widget.ts` | 69 | Beginner | setWidget, TUI Text component, ANSI backgrounds |
| `extensions/theme-cycler.ts` | 182 | Intermediate | Shortcuts, commands, timers, setStatus |
| `extensions/tool-counter.ts` | 103 | Intermediate | Multi-line footer, session traversal, git branch |
| `extensions/cross-agent.ts` | 265 | Intermediate | Dynamic commands, frontmatter, expandArgs |
| `extensions/system-select.ts` | 168 | Intermediate | System prompt swap, setActiveTools, select dialog |
| `extensions/session-replay.ts` | 217 | Intermediate | ctx.ui.custom, overlay UI, keyboard navigation |
| `extensions/damage-control.ts` | 206 | Intermediate | tool_call blocking, isToolCallEventType, YAML rules |
| `extensions/tilldone.ts` | 727 | Advanced | registerTool, TypeBox, state reconstruction, CQRS |
| `extensions/subagent-widget.ts` | 482 | Advanced | Subprocess spawning, JSONL streaming, session files |
| `extensions/agent-chain.ts` | 797 | Advanced | Pipeline orchestration, YAML chains, $INPUT substitution |
| `extensions/agent-team.ts` | 734 | Advanced | Dispatcher pattern, teams.yaml, grid dashboard |
| `extensions/pi-pi.ts` | 633 | Advanced | Parallel research, Promise.allSettled, expert agents |
| `CLAUDE.md` | 21 | Reference | Project conventions and tooling summary |
| `COMPARISON.md` | 244 | Reference | Pi vs Claude Code feature comparison |
| `QUICKSTART.md` | ~1045 | Reference | Setup guide, use cases, troubleshooting |
| `THEME.md` | 30 | Reference | Semantic color token roles and examples |
| `TOOLS.md` | 27 | Reference | Built-in tool function signatures |
| `RESERVED_KEYS.md` | 76 | Reference | Pi reserved keys and safe extension keys |
| `.pi/agents/*.md` | 5–10 each | Reference | Agent personas with frontmatter tool restrictions |
| `.pi/agents/teams.yaml` | 32 | Reference | Team definitions mapping names to agent lists |
| `.pi/agents/agent-chain.yaml` | 50 | Reference | Pipeline chain definitions with step prompts |
| `.pi/damage-control-rules.yaml` | ~280 | Reference | Safety rule patterns by category |
| `justfile` | 107 | Reference | All runnable recipes and extension combinations |
| `package.json` | 9 | Reference | Project dependencies (just `yaml`) |
| `.pi/settings.json` | 4 | Reference | Workspace theme and prompt path configuration |

---

## Appendix D: Common Mistakes and How to Avoid Them

**Mistake 1: Registering event handlers inside other event handlers**

```typescript
// WRONG — pi.on inside session_start
pi.on("session_start", async (_event, ctx) => {
    pi.on("tool_call", async (event) => { ... });  // registers a new handler EVERY session
});

// CORRECT — both at top level
pi.on("session_start", async (_event, ctx) => { ... });
pi.on("tool_call", async (event) => { ... });
```

Each `pi.on` call adds a new handler to the list. Calling it inside `session_start`
adds a new handler on every session restart, causing duplicate handlers and memory leaks.

**Mistake 2: Forgetting `as const` on return values**

```typescript
// WRONG — TypeScript infers string, not the literal type
return { action: "handled" };

// CORRECT
return { action: "handled" as const };
```

**Mistake 3: Using `visibleWidth` on content that contains no ANSI codes**

The function handles both cases correctly, but calling it unnecessarily on a plain
string is harmless. The important thing is to *always* use it when ANSI codes are
present — forgetting it causes the footer layout to drift.

**Mistake 4: Not cleaning up timers**

Any `setInterval` or `setTimeout` that closes over `ctx` will cause an error if it fires
after the session has ended. Always clear timers in `session_shutdown`.

**Mistake 5: Using `import` instead of `import type` for type-only imports**

```typescript
// WRONG — imports the runtime module even though only the type is used
import { ExtensionAPI } from "@mariozechner/pi-coding-agent";

// CORRECT — erased at compile time, no runtime cost
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
```

**Mistake 6: Blocking the `session_start` handler with an awaited dialog**

```typescript
// WRONG — blocks session_start until the dialog is closed
pi.on("session_start", async (_event, ctx) => {
    const answer = await ctx.ui.input("What is your purpose?", "");
});

// CORRECT — fire-and-forget, session_start returns immediately
pi.on("session_start", async (_event, ctx) => {
    void askForPurpose(ctx);  // starts the async function without awaiting it
});
```

---

## Appendix E: External Documentation Links

For deeper reading beyond this codebase:

| Resource | URL |
|----------|-----|
| Pi extension system API | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md |
| Pi TypeScript SDK reference | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/sdk.md |
| Pi JSON event stream format | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/json.md |
| Pi providers and models | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/providers.md |
| Pi settings reference | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/settings.md |
| TypeBox schema library | https://github.com/sinclairzx81/typebox |
| TypeScript handbook | https://www.typescriptlang.org/docs/handbook/intro.html |
| just task runner | https://just.systems/man/en/ |
| Bun runtime | https://bun.sh/docs |
