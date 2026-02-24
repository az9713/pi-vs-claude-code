# Codebase Tutorial — Pi vs Claude Code Extension Playground

A step-by-step walkthrough of every file, pattern, and concept in this repository.
This tutorial assumes zero familiarity with the project. By the end, you will
understand how every extension works, how they compose, and how the multi-agent
orchestration patterns build on each other.

---

## Table of Contents

- [Step 1: Orientation — What This Project Is](#step-1-orientation--what-this-project-is)
- [Step 2: Project Structure at a Glance](#step-2-project-structure-at-a-glance)
- [Step 3: Tooling and How to Run Things](#step-3-tooling-and-how-to-run-things)
- [Step 4: The Shared Foundation — themeMap.ts](#step-4-the-shared-foundation--thememapts)
- [Step 5: Your First Extension — pure-focus.ts](#step-5-your-first-extension--pure-focusts)
- [Step 6: A Compact Footer — minimal.ts](#step-6-a-compact-footer--minimalts)
- [Step 7: Rich Footer With Live Data — tool-counter.ts](#step-7-rich-footer-with-live-data--tool-counterts)
- [Step 8: Widgets Above the Editor — tool-counter-widget.ts](#step-8-widgets-above-the-editor--tool-counter-widgetts)
- [Step 9: Keyboard Shortcuts and Commands — theme-cycler.ts](#step-9-keyboard-shortcuts-and-commands--theme-cyclerts)
- [Step 10: Intercepting User Input — purpose-gate.ts](#step-10-intercepting-user-input--purpose-gatets)
- [Step 11: Cross-Agent Discovery — cross-agent.ts](#step-11-cross-agent-discovery--cross-agentts)
- [Step 12: Switching System Prompts — system-select.ts](#step-12-switching-system-prompts--system-selectts)
- [Step 13: Blocking Tool Calls — damage-control.ts](#step-13-blocking-tool-calls--damage-controlts)
- [Step 14: Custom Tools and Task Discipline — tilldone.ts](#step-14-custom-tools-and-task-discipline--tilldonets)
- [Step 15: Spawning Background Subagents — subagent-widget.ts](#step-15-spawning-background-subagents--subagent-widgetts)
- [Step 16: Full-Screen Overlays — session-replay.ts](#step-16-full-screen-overlays--session-replayts)
- [Step 17: Multi-Agent Teams — agent-team.ts](#step-17-multi-agent-teams--agent-teamts)
- [Step 18: Sequential Pipelines — agent-chain.ts](#step-18-sequential-pipelines--agent-chaints)
- [Step 19: Parallel Expert Research — pi-pi.ts](#step-19-parallel-expert-research--pi-pits)
- [Step 20: Agent Definitions and Team Configuration](#step-20-agent-definitions-and-team-configuration)
- [Step 21: The Theme System](#step-21-the-theme-system)
- [Step 22: Safety Rules — damage-control-rules.yaml](#step-22-safety-rules--damage-control-rulesyaml)
- [Step 23: How Extensions Compose](#step-23-how-extensions-compose)
- [Step 24: Patterns Cheat Sheet](#step-24-patterns-cheat-sheet)

---

## Step 1: Orientation — What This Project Is

This repository is a collection of **16 Pi Coding Agent extensions** plus their
supporting configuration files (agents, themes, safety rules, team definitions).

**Pi Coding Agent** is an open-source, terminal-based AI coding assistant. It works
like Claude Code or Cursor's terminal mode: you type a task in plain English, and
the AI reads files, writes code, runs shell commands, and reports results — all
inside your terminal.

What makes Pi different is its **in-process extension system**. Extensions are
TypeScript files that run inside Pi's own runtime. They can:

- Replace the footer, header, or any UI element
- Add widgets above or below the editor
- Register new slash commands (`/theme`, `/sub`, `/tilldone`)
- Register keyboard shortcuts (`Ctrl+X`, `Ctrl+Q`)
- Register custom tools the AI can call (like `tilldone` or `dispatch_agent`)
- Intercept and block tool calls before they execute
- Modify the system prompt that shapes AI behavior
- Spawn child Pi processes as background workers

Each extension in this repo demonstrates one or more of these capabilities,
building from the simplest (removing the footer) to the most complex (parallel
multi-agent orchestration).

---

## Step 2: Project Structure at a Glance

```
pi-vs-claude-code/
├── extensions/                # 15 extension files + 1 shared module
│   ├── themeMap.ts            # Shared: theme assignment + title helper
│   ├── pure-focus.ts          # Removes all footer/status UI
│   ├── minimal.ts             # Model name + 10-block context bar
│   ├── tool-counter.ts        # Rich 2-line footer with tokens/cost/tools
│   ├── tool-counter-widget.ts # Tool counts as colored badges in a widget
│   ├── theme-cycler.ts        # Ctrl+X/Q to cycle themes, /theme picker
│   ├── purpose-gate.ts        # Forces a declared purpose before work
│   ├── cross-agent.ts         # Discovers commands from .claude/.gemini/.codex
│   ├── system-select.ts       # /system to pick an agent persona
│   ├── damage-control.ts      # Blocks dangerous commands per YAML rules
│   ├── tilldone.ts            # Task-driven discipline with blocking gate
│   ├── subagent-widget.ts     # /sub spawns background workers with live UI
│   ├── session-replay.ts      # /replay scrollable session timeline overlay
│   ├── agent-team.ts          # Multi-agent dispatcher with grid dashboard
│   ├── agent-chain.ts         # Sequential pipeline orchestrator
│   └── pi-pi.ts               # Meta-agent with parallel expert research
│
├── .pi/
│   ├── settings.json          # Pi config: default theme, prompt paths
│   ├── damage-control-rules.yaml  # Safety rules for damage-control.ts
│   ├── agents/                # Agent persona definitions (.md frontmatter)
│   │   ├── scout.md           # Read-only codebase explorer
│   │   ├── planner.md         # Architecture/planning specialist
│   │   ├── builder.md         # Implementation agent (has write tools)
│   │   ├── reviewer.md        # Code review agent
│   │   ├── documenter.md      # Documentation writer
│   │   ├── red-team.md        # Security/adversarial tester
│   │   ├── plan-reviewer.md   # Plan critic
│   │   ├── bowser.md          # Headless browser automation
│   │   ├── teams.yaml         # Team compositions for agent-team.ts
│   │   ├── agent-chain.yaml   # Pipeline definitions for agent-chain.ts
│   │   └── pi-pi/             # Expert agents for the pi-pi meta-agent
│   │       ├── pi-orchestrator.md
│   │       ├── ext-expert.md
│   │       ├── theme-expert.md
│   │       ├── tui-expert.md
│   │       ├── config-expert.md
│   │       ├── skill-expert.md
│   │       ├── prompt-expert.md
│   │       ├── agent-expert.md
│   │       ├── cli-expert.md
│   │       └── keybinding-expert.md
│   └── themes/                # 11 custom JSON theme files
│       ├── synthwave.json
│       ├── dracula.json
│       ├── tokyo-night.json
│       └── ... (8 more)
│
├── docs/                      # Documentation
│   ├── QUICKSTART.md          # Setup guide + 10 hands-on use cases
│   ├── ARCHITECTURE.md        # Full architecture with diagrams
│   ├── DEVELOPER_GUIDE.md     # API reference + extension tutorials
│   ├── STUDY_PLAN.md          # 9-phase zero-to-hero learning plan
│   └── tutorials/             # 10 step-by-step build tutorials
│
├── specs/                     # Feature specifications
├── justfile                   # Task runner recipes (just ext-minimal, etc.)
├── package.json               # Only dependency: yaml
├── CLAUDE.md                  # Project instructions for AI agents
├── COMPARISON.md              # Claude Code vs Pi feature matrix
├── THEME.md                   # Color token conventions
├── TOOLS.md                   # Built-in tool signatures
└── RESERVED_KEYS.md           # Keyboard shortcuts Pi owns vs free for extensions
```

---

## Step 3: Tooling and How to Run Things

### Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| **bun** | JS/TS runtime + package manager | `curl -fsSL https://bun.sh/install \| bash` |
| **just** | Task runner (reads `justfile`) | `brew install just` or `cargo install just` |
| **pi** | The AI coding agent itself | `npm install -g @mariozechner/pi-coding-agent` |

### Setup

```bash
# 1. Install dependencies (just the yaml package)
bun install

# 2. Copy and fill in your API keys
cp .env.sample .env
# Edit .env with your ANTHROPIC_API_KEY, OPENROUTER_API_KEY, etc.

# 3. List all available recipes
just
```

### Running Extensions

The `justfile` wraps every extension into a single command. The first line
`set dotenv-load := true` auto-loads your `.env` file:

```bash
just pi                    # Plain Pi, no extensions
just ext-minimal           # Minimal footer (context bar)
just ext-tool-counter      # Rich 2-line footer
just ext-agent-team        # Multi-agent dispatcher
```

Under the hood, each recipe calls `pi -e extensions/<name>.ts`. Multiple `-e`
flags stack extensions:

```bash
# This is what `just ext-minimal` actually runs:
pi -e extensions/minimal.ts -e extensions/theme-cycler.ts
```

---

## Step 4: The Shared Foundation — themeMap.ts

**File:** `extensions/themeMap.ts` (143 lines)

This is not an extension itself — it is a shared utility module imported by every
extension. It provides two things:

### 1. Per-Extension Theme Assignment

Each extension maps to one of the 11 custom themes in `.pi/themes/`:

```typescript
export const THEME_MAP: Record<string, string> = {
    "minimal":         "synthwave",
    "agent-team":      "dracula",
    "damage-control":  "gruvbox",
    "pi-pi":           "rose-pine",
    // ... etc
};
```

When multiple extensions are stacked (e.g., `minimal` + `theme-cycler`), only
the **primary** extension (first `-e` flag) gets to set the theme. This is
determined by reading `process.argv`:

```typescript
function primaryExtensionName(): string | null {
    const argv = process.argv;
    for (let i = 0; i < argv.length - 1; i++) {
        if (argv[i] === "-e" || argv[i] === "--extension") {
            return basename(argv[i + 1]).replace(/\.[^.]+$/, "");
        }
    }
    return null;
}
```

### 2. Terminal Title

Every extension sets the terminal tab title to `π - <extension-name>`. The title
is deferred 150ms to fire after Pi's own startup title-set:

```typescript
function applyExtensionTitle(ctx: ExtensionContext): void {
    const name = primaryExtensionName();
    if (!name) return;
    setTimeout(() => ctx.ui.setTitle(`π - ${name}`), 150);
}
```

### Combined Entry Point

Every extension calls this single function in its `session_start` handler:

```typescript
applyExtensionDefaults(import.meta.url, ctx);
```

This applies both theme and title. The `import.meta.url` is how the function
knows which extension file is calling it.

**Key takeaway:** This module solves the "stacking problem" — when N extensions
are loaded, they all fire `session_start`, but only the primary one controls
theme and title.

---

## Step 5: Your First Extension — pure-focus.ts

**File:** `extensions/pure-focus.ts` (24 lines)
**Run:** `just ext-pure-focus`

This is the simplest possible extension. It removes the entire footer:

```typescript
export default function (pi: ExtensionAPI) {
    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        ctx.ui.setFooter((_tui, _theme, _footerData) => ({
            dispose: () => {},
            invalidate() {},
            render(_width: number): string[] {
                return [];  // Empty array = no footer lines
            },
        }));
    });
}
```

### What This Teaches

1. **Extension signature:** Every extension exports a default function that
   receives `pi: ExtensionAPI`.
2. **Event listeners:** `pi.on("session_start", handler)` registers a callback
   for when a session begins.
3. **Footer API:** `ctx.ui.setFooter()` takes a factory function returning an
   object with `dispose()`, `invalidate()`, and `render(width)`. The `render`
   method returns an array of strings — one per line. Returning `[]` means no
   footer at all.

### Exercise

Try modifying the render function to return `["Hello from my extension!"]` and
run `just ext-pure-focus` to see your message in the footer.

---

## Step 6: A Compact Footer — minimal.ts

**File:** `extensions/minimal.ts` (33 lines)
**Run:** `just ext-minimal`

Builds on pure-focus by showing a single-line footer with the model name and a
10-block context usage bar:

```
claude-sonnet-4-5        [###-------] 30%
```

### Key Code

```typescript
render(width: number): string[] {
    const model = ctx.model?.id || "no-model";
    const usage = ctx.getContextUsage();
    const pct = (usage && usage.percent !== null) ? usage.percent : 0;
    const filled = Math.round(pct / 10);
    const bar = "#".repeat(filled) + "-".repeat(10 - filled);

    const left = theme.fg("dim", ` ${model}`);
    const right = theme.fg("dim", `[${bar}] ${Math.round(pct)}% `);
    const pad = " ".repeat(Math.max(1, width - visibleWidth(left) - visibleWidth(right)));

    return [truncateToWidth(left + pad + right, width)];
}
```

### What This Teaches

1. **Context usage:** `ctx.getContextUsage()` returns the current context window
   fill percentage. This is how you know how much "memory" the AI has used.
2. **Theme colors:** `theme.fg("dim", text)` wraps text in ANSI color codes using
   the current theme's `dim` color token. The five standard tokens are:
   - `success` (green) — primary values
   - `accent` (cyan) — secondary values
   - `warning` (yellow/orange) — punctuation, framing
   - `dim` — filler, labels
   - `muted` — subdued text
3. **Width management:** `visibleWidth()` counts only visible characters (ignoring
   ANSI escape codes). `truncateToWidth()` truncates to fit the terminal width.
   These are essential for footer rendering — terminals are fixed-width.

### Exercise

Change the bar from `#`/`-` characters to `█`/`░` for a solid block style. Run
it and compare.

---

## Step 7: Rich Footer With Live Data — tool-counter.ts

**File:** `extensions/tool-counter.ts` (102 lines)
**Run:** `just ext-tool-counter`

A two-line footer that displays real-time token usage, cost, git branch, and
per-tool call counts:

```
claude-sonnet-4-5  [###-------] 30%          12.4k in  2.1k out  $0.0032
my-project (main)                  read 4 | bash 2 | grep 1
```

### Key Patterns

**Tool call counting via event listener:**

```typescript
const counts: Record<string, number> = {};

pi.on("tool_execution_end", async (event) => {
    counts[event.toolName] = (counts[event.toolName] || 0) + 1;
});
```

The `tool_execution_end` event fires after every tool call. The counts map
accumulates per-tool totals.

**Session history traversal for token/cost totals:**

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

`ctx.sessionManager.getBranch()` returns the full message history for the current
branch. Each assistant message carries `.usage` with token counts and cost.

**Git branch tracking:**

```typescript
ctx.ui.setFooter((tui, theme, footerData) => {
    const unsub = footerData.onBranchChange(() => tui.requestRender());
    // ...
    const branch = footerData.getGitBranch();
    return { dispose: unsub, /* ... */ };
});
```

`footerData.onBranchChange()` subscribes to git branch changes. The returned
`unsub` function is passed to `dispose()` for cleanup.

### What This Teaches

- Event-driven state accumulation (`tool_execution_end`)
- Session history traversal for aggregated metrics
- Multi-line footer rendering
- Resource cleanup via `dispose()`

---

## Step 8: Widgets Above the Editor — tool-counter-widget.ts

**File:** `extensions/tool-counter-widget.ts` (68 lines)
**Run:** `just ext-tool-counter-widget`

Instead of the footer, this shows tool counts as colored badge widgets above
the editor:

```
Tools (7): [Bash 3] [Read 4]
```

### Key Code

```typescript
ctx.ui.setWidget("tool-counter", (_tui, theme) => {
    const text = new Text("", 1, 1);
    return {
        render(width: number): string[] {
            // Build colored badges with ANSI background colors
            const parts = entries.map(([name, count]) => {
                const rgb = toolColors[name];
                return bg(rgb, `\x1b[38;2;220;220;220m  ${name} ${count}  \x1b[39m`);
            });
            text.setText(/* assembled text */);
            return text.render(width);
        },
        invalidate() { text.invalidate(); },
    };
});
```

### What This Teaches

1. **Widget API:** `ctx.ui.setWidget(id, factory)` registers a named widget.
   The factory returns `render()` and `invalidate()`.
2. **Raw ANSI codes:** For effects beyond theme colors (like background fills),
   you can write ANSI escape sequences directly:
   `\x1b[48;2;R;G;Bm` = set background RGB, `\x1b[49m` = reset background.
3. **TUI components:** `Text` from `@mariozechner/pi-tui` is a layout-aware
   text component. Call `setText()` to update content, `render(width)` to get
   the output lines.

---

## Step 9: Keyboard Shortcuts and Commands — theme-cycler.ts

**File:** `extensions/theme-cycler.ts` (181 lines)
**Run:** `just ext-theme-cycler`

Adds keyboard shortcuts and a slash command for cycling through themes:

- `Ctrl+X` — cycle forward
- `Ctrl+Q` — cycle backward
- `/theme` — open a select picker
- `/theme <name>` — switch directly

### Key Patterns

**Registering shortcuts:**

```typescript
pi.registerShortcut("ctrl+x", {
    description: "Cycle theme forward",
    handler: async (ctx) => {
        cycleTheme(ctx, 1);
    },
});
```

**Registering commands:**

```typescript
pi.registerCommand("theme", {
    description: "Select a theme: /theme or /theme <name>",
    handler: async (args, ctx) => {
        // args contains everything after "/theme "
        if (args.trim()) {
            ctx.ui.setTheme(args.trim());
        } else {
            const selected = await ctx.ui.select("Select Theme", items);
            // ...
        }
    },
});
```

**Temporary "swatch" widget with auto-dismiss:**

```typescript
ctx.ui.setWidget("theme-swatch", factory, { placement: "belowEditor" });

swatchTimer = setTimeout(() => {
    ctx.ui.setWidget("theme-swatch", undefined);  // Remove widget
}, 3000);
```

Setting a widget to `undefined` removes it. The swatch appears for 3 seconds
after each theme change, then vanishes.

### What This Teaches

- `pi.registerShortcut()` — keybindings (see `RESERVED_KEYS.md` for which keys are available)
- `pi.registerCommand()` — slash commands with argument parsing
- `ctx.ui.select()` — interactive selection dialogs
- `ctx.ui.setTheme()` — programmatic theme switching
- `ctx.ui.setStatus()` — status line updates
- Widget lifecycle: create, update, and destroy

---

## Step 10: Intercepting User Input — purpose-gate.ts

**File:** `extensions/purpose-gate.ts` (84 lines)
**Run:** `just ext-purpose-gate`

Forces the user to declare a purpose before any work begins. Shows a persistent
banner widget and injects the purpose into the system prompt.

### Three Key Mechanisms

**1. Input dialog on startup:**

```typescript
const answer = await ctx.ui.input(
    "What is the purpose of this agent?",
    "e.g. Refactor the auth module to use JWT"
);
```

`ctx.ui.input()` blocks until the user types an answer. The loop continues
until a non-empty answer is given.

**2. Input blocking:**

```typescript
pi.on("input", async (_event, ctx) => {
    if (!purpose) {
        ctx.ui.notify("Set a purpose first.", "warning");
        return { action: "handled" as const };
    }
    return { action: "continue" as const };
});
```

Returning `{ action: "handled" }` from the `input` event handler consumes the
input — it never reaches the AI. This is the "gate" pattern.

**3. System prompt injection:**

```typescript
pi.on("before_agent_start", async (event) => {
    if (!purpose) return;
    return {
        systemPrompt: event.systemPrompt +
            `\n\n<purpose>\nYour singular purpose: ${purpose}\n</purpose>`,
    };
});
```

The `before_agent_start` event fires before each AI turn. Returning
`{ systemPrompt }` replaces the system prompt entirely. Here it appends the
purpose to the existing prompt.

### What This Teaches

- `ctx.ui.input()` — text input dialogs
- `pi.on("input")` — intercepting user messages
- `pi.on("before_agent_start")` — modifying the system prompt
- Combining UI gates with prompt engineering

---

## Step 11: Cross-Agent Discovery — cross-agent.ts

**File:** `extensions/cross-agent.ts` (265 lines)
**Run:** `just ext-cross-agent`

Scans directories from other AI coding tools (`.claude/`, `.gemini/`, `.codex/`)
for commands, skills, and agent definitions. Registers discovered commands as
Pi slash commands.

### How It Works

1. On `session_start`, scans both project-local and global (`~/`) directories
   for each provider
2. Parses `.md` files with frontmatter for commands, skills, and agents
3. Registers each discovered command using `pi.registerCommand()`
4. Displays a formatted notification showing what was found

### Key Pattern — Frontmatter Parsing

Agent definitions use YAML frontmatter in Markdown files:

```markdown
---
name: scout
description: Fast recon and codebase exploration
tools: read,grep,find,ls
---
You are a scout agent. Investigate the codebase...
```

The parser extracts fields between `---` markers:

```typescript
function parseFrontmatter(raw: string) {
    const match = raw.match(/^---\s*\n([\s\S]*?)\n---\s*\n([\s\S]*)$/);
    // match[1] = frontmatter YAML, match[2] = body
}
```

### Template Expansion

Discovered commands support argument substitution when invoked:

```typescript
function expandArgs(template: string, args: string): string {
    result = result.replace(/\$ARGUMENTS|\$@/g, args);  // All args
    result = result.replaceAll(`$${i + 1}`, parts[i]);  // Positional: $1, $2
}
```

---

## Step 12: Switching System Prompts — system-select.ts

**File:** `extensions/system-select.ts` (167 lines)
**Run:** `just ext-system-select`

The `/system` command opens a picker showing every agent definition from
`.pi/agents/`, `.claude/agents/`, etc. Selecting one:

1. Prepends the agent's body to Pi's system prompt
2. Restricts tools to the agent's declared tool set (if specified)

### Key Code

```typescript
pi.on("before_agent_start", async (event, _ctx) => {
    if (!activeAgent) return;
    return {
        systemPrompt: activeAgent.body + "\n\n" + event.systemPrompt,
    };
});
```

**Tool restriction:**

```typescript
if (agent.tools.length > 0) {
    pi.setActiveTools(agent.tools);  // Only these tools available
} else {
    pi.setActiveTools(defaultTools);  // Reset to full tool set
}
```

`pi.setActiveTools()` is a hard restriction — the AI literally cannot call tools
not in the list. Combined with system prompt injection, this transforms Pi into
a completely different agent persona.

---

## Step 13: Blocking Tool Calls — damage-control.ts

**File:** `extensions/damage-control.ts` (206 lines)
**Run:** `just ext-damage-control`

Intercepts every tool call and checks it against safety rules loaded from
`.pi/damage-control-rules.yaml`. Dangerous operations are blocked or require
user confirmation.

### The tool_call Event

```typescript
pi.on("tool_call", async (event, ctx) => {
    // Check against rules...
    if (violationReason) {
        if (shouldAsk) {
            const confirmed = await ctx.ui.confirm(/*...*/);
            if (!confirmed) {
                ctx.abort();
                return { block: true, reason: `BLOCKED: ${violationReason}` };
            }
        } else {
            ctx.abort();
            return { block: true, reason: `BLOCKED: ${violationReason}` };
        }
    }
    return { block: false };
});
```

### What `block: true` Does

When a `tool_call` handler returns `{ block: true, reason }`:
- The tool **does not execute**
- The `reason` string is sent back to the AI as the tool result
- The AI sees "BLOCKED" and (ideally) adjusts its approach

### Type-Safe Event Narrowing

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

if (isToolCallEventType("bash", event)) {
    const command = event.input.command;  // TypeScript knows this field exists
}
if (isToolCallEventType("read", event)) {
    const path = event.input.path;  // Different shape, different fields
}
```

`isToolCallEventType()` is a type guard that narrows the event to a specific
tool type, giving you typed access to that tool's input parameters.

### Four Rule Categories

| Category | What it protects |
|----------|-----------------|
| `bashToolPatterns` | RegExp patterns matched against bash commands |
| `zeroAccessPaths` | Paths that cannot be read, written, or accessed at all |
| `readOnlyPaths` | Paths that can be read but not modified |
| `noDeletePaths` | Paths that cannot be deleted or moved |

---

## Step 14: Custom Tools and Task Discipline — tilldone.ts

**File:** `extensions/tilldone.ts` (726 lines)
**Run:** `just ext-tilldone`

The most complex single extension. It registers a custom tool (`tilldone`) that
the AI must use to define tasks before it can use any other tools.

### Three Interconnected Systems

**1. Custom Tool Registration:**

```typescript
pi.registerTool({
    name: "tilldone",
    label: "TillDone",
    description: "Manage your task list. You MUST add tasks before using any other tools.",
    parameters: TillDoneParams,  // TypeBox schema

    async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
        switch (params.action) {
            case "add": /* ... */
            case "toggle": /* ... */
            case "list": /* ... */
            // ...
        }
    },

    renderCall(args, theme) { /* Custom tool call display */ },
    renderResult(result, { expanded }, theme) { /* Custom result display */ },
});
```

Tools have:
- `parameters` — a TypeBox schema defining the JSON arguments
- `execute()` — the function that runs when the AI calls the tool
- `renderCall()` — how the tool call appears in the conversation UI
- `renderResult()` — how the result appears (supports expanded/collapsed states)

**2. Blocking Gate:**

```typescript
pi.on("tool_call", async (event, _ctx) => {
    if (event.toolName === "tilldone") return { block: false };  // Always allow

    if (tasks.length === 0) {
        return {
            block: true,
            reason: "No tasks defined. Use tilldone add first!",
        };
    }
    // ... more checks
    return { block: false };
});
```

All non-tilldone tools are blocked until tasks exist and one is marked
"in progress." This enforces plan-before-execute discipline.

**3. State Reconstruction:**

```typescript
const reconstructState = (ctx: ExtensionContext) => {
    tasks = [];
    for (const entry of ctx.sessionManager.getBranch()) {
        if (entry.type !== "message") continue;
        if (msg.role !== "toolResult" || msg.toolName !== "tilldone") continue;
        const details = msg.details as TillDoneDetails | undefined;
        if (details) {
            tasks = details.tasks;
            nextId = details.nextId;
        }
    }
};
```

When the user navigates session branches (undo/redo), the extension rebuilds
its state by scanning session history for previous `tilldone` tool results.
Each tool result carries a `details` object with the full task list snapshot.

### UI Surfaces

TillDone renders to **four** different UI surfaces simultaneously:
- **Footer:** Task progress bar and recent task list
- **Widget:** "WORKING ON #3 — Implement auth module" banner
- **Status line:** Compact "4 tasks (2 remaining)" summary
- **Overlay:** `/tilldone` opens a full interactive task viewer

---

## Step 15: Spawning Background Subagents — subagent-widget.ts

**File:** `extensions/subagent-widget.ts` (481 lines)
**Run:** `just ext-subagent-widget`

Spawns background Pi processes with live-updating progress widgets.

### The Fire-and-Forget Pattern

```typescript
const proc = spawn("pi", [
    "--mode", "json",      // Output JSONL events to stdout
    "-p",                  // Prompt mode (non-interactive)
    "--session", state.sessionFile,  // Persistent session file
    "--no-extensions",
    "--model", model,
    "--tools", "read,bash,grep,find,ls",
    prompt,
], {
    stdio: ["ignore", "pipe", "pipe"],
});
```

Key flags:
- `--mode json` — outputs JSONL (one JSON event per line) to stdout
- `-p` — runs in prompt mode (submits prompt, exits when done)
- `--session` — saves conversation to a file for later resumption

### Streaming Event Processing

```typescript
proc.stdout.on("data", (chunk: string) => {
    buffer += chunk;
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";  // Keep incomplete last line
    for (const line of lines) {
        const event = JSON.parse(line);
        if (event.type === "message_update") {
            state.textChunks.push(event.assistantMessageEvent.delta);
        } else if (event.type === "tool_execution_start") {
            state.toolCount++;
        }
    }
});
```

The subprocess emits JSONL events. The parent parses them line by line using a
buffer (since `data` events may split across JSON boundaries).

### Result Delivery

```typescript
pi.sendMessage({
    customType: "subagent-result",
    content: `Subagent #${state.id} finished. Result:\n${result}`,
    display: true,
}, { deliverAs: "followUp", triggerTurn: true });
```

`pi.sendMessage()` with `triggerTurn: true` injects a message into the main
conversation and triggers the AI to respond to it. This is how background results
get "delivered" to the primary agent.

### Conversation Continuations

The `/subcont` command continues an existing subagent's conversation by reusing
the same `--session` file. The subagent picks up where it left off with full
context of its previous work.

### Custom Tools for AI-Driven Orchestration

This extension also registers tools (`subagent_create`, `subagent_continue`,
`subagent_remove`, `subagent_list`) so the AI itself can spawn and manage
subagents programmatically — not just the human via slash commands.

---

## Step 16: Full-Screen Overlays — session-replay.ts

**File:** `extensions/session-replay.ts` (216 lines)
**Run:** `just ext-session-replay`

The `/replay` command opens a scrollable, keyboard-navigable overlay showing
the full session timeline.

### Custom UI Component

```typescript
await ctx.ui.custom((tui, theme, kb, done) => {
    const component = new SessionReplayUI(items, () => done(undefined));
    return {
        render: (w) => component.render(w, 30, theme),
        handleInput: (data) => component.handleInput(data, tui),
        invalidate: () => {},
    };
}, {
    overlay: true,
    overlayOptions: { width: "80%", anchor: "center" },
});
```

`ctx.ui.custom()` takes the full screen and renders a custom component until
`done()` is called. The component must implement:
- `render(width)` — returns string lines
- `handleInput(data)` — handles keyboard input
- `invalidate()` — signals state change

### Keyboard Navigation

```typescript
handleInput(data: string): void {
    if (matchesKey(data, Key.up)) {
        this.selectedIndex = Math.max(0, this.selectedIndex - 1);
    } else if (matchesKey(data, Key.enter)) {
        this.expandedIndex = this.expandedIndex === this.selectedIndex
            ? null : this.selectedIndex;
    } else if (matchesKey(data, Key.escape)) {
        this.onClose();
    }
}
```

`matchesKey()` from `@mariozechner/pi-tui` handles cross-terminal key detection.
`Key.up`, `Key.down`, `Key.enter`, `Key.escape` are predefined constants.

### TUI Components Used

- `Container` — vertical layout container
- `Text` — text block with ANSI support
- `Box` — bordered box with optional background
- `Spacer` — vertical spacing
- `DynamicBorder` — horizontal rule with custom styling
- `Markdown` — renders Markdown with syntax highlighting

---

## Step 17: Multi-Agent Teams — agent-team.ts

**File:** `extensions/agent-team.ts` (734 lines)
**Run:** `just ext-agent-team`

This is the first orchestration extension. The primary Pi agent becomes a
**dispatcher** — it has no codebase tools and can only delegate work to specialist
agents via the `dispatch_agent` tool.

### Architecture

```
User → Dispatcher (primary Pi) → dispatch_agent tool → Pi subprocess
                                                         ↓
                                                   Specialist agent
                                                   (scout/builder/etc.)
                                                         ↓
                                                   Result returned
```

### Key Design Decisions

**1. Tool lockdown:**

```typescript
pi.setActiveTools(["dispatch_agent"]);
```

The dispatcher can ONLY call `dispatch_agent`. It cannot read files, run bash,
or write code directly. All work must go through specialist agents.

**2. Team loading from YAML:**

```typescript
// .pi/agents/teams.yaml
full:
  - scout
  - planner
  - builder
  - reviewer
  - documenter
  - red-team

plan-build:
  - planner
  - builder
  - reviewer
```

Teams define which agents are available. The `/agents-team` command switches
between them.

**3. System prompt override:**

```typescript
pi.on("before_agent_start", async () => {
    const agentCatalog = Array.from(agentStates.values())
        .map(s => `### ${s.def.name}\n${s.def.description}\nTools: ${s.def.tools}`)
        .join("\n\n");

    return {
        systemPrompt: `You are a dispatcher agent. You coordinate specialists.
You do NOT have direct codebase access. ALWAYS use dispatch_agent.

## Agents\n\n${agentCatalog}`,
    };
});
```

The entire system prompt is replaced with dispatcher instructions and a catalog
of available agents.

**4. Grid dashboard widget:**

Each agent gets a card in a responsive grid showing status, elapsed time,
context usage, and last work preview. The grid auto-sizes based on team size.

**5. Session persistence:**

Each specialist agent gets its own session file in `.pi/agent-sessions/`. On
re-dispatch, the agent resumes with full context of its previous work (using
the `-c` flag for "continue").

---

## Step 18: Sequential Pipelines — agent-chain.ts

**File:** `extensions/agent-chain.ts` (797 lines)
**Run:** `just ext-agent-chain`

Where agent-team lets the dispatcher choose which agents to use, agent-chain
defines **fixed sequences** (pipelines) where each step's output feeds the next.

### Pipeline Definition

```yaml
# .pi/agents/agent-chain.yaml
plan-build-review:
  description: "Plan, implement, and review"
  steps:
    - agent: planner
      prompt: "Plan the implementation for: $INPUT"
    - agent: builder
      prompt: "Implement the following plan:\n\n$INPUT"
    - agent: reviewer
      prompt: "Review this implementation:\n\n$INPUT"
```

### Variable Substitution

- `$INPUT` — output from the previous step (or the user's prompt for step 1)
- `$ORIGINAL` — always the user's original prompt, unchanged

```typescript
const resolvedPrompt = step.prompt
    .replace(/\$INPUT/g, input)
    .replace(/\$ORIGINAL/g, originalPrompt);
```

### Sequential Execution

```typescript
for (let i = 0; i < activeChain.steps.length; i++) {
    stepStates[i].status = "running";
    const result = await runAgent(agentDef, resolvedPrompt, i, ctx);
    stepStates[i].status = result.exitCode === 0 ? "done" : "error";
    input = result.output;  // Feed output to next step
}
```

### Visual Pipeline Widget

Step cards are rendered horizontally with arrows between them:

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│ Planner  │ ──▶   │ Builder  │ ──▶   │ Reviewer  │
│ ✓ done 8s│       │ ● run 4s │       │ ○ pending │
│ Plan the…│       │ Building…│       │ —         │
└──────────┘       └──────────┘       └──────────┘
```

### Hybrid Model

Unlike agent-team (dispatcher-only), agent-chain gives the primary agent **both**
its normal tools AND the `run_chain` tool. The system prompt instructs it:
- Use `run_chain` for significant work (features, refactors, multi-file changes)
- Work directly for simple tasks (reading a file, answering questions)

---

## Step 19: Parallel Expert Research — pi-pi.ts

**File:** `extensions/pi-pi.ts` (633 lines)
**Run:** `just ext-pi-pi`

The most advanced extension. A meta-agent that builds Pi extensions by querying
domain-specific expert subagents **in parallel**.

### Architecture

```
User: "Build an extension that shows a live clock"
  │
  ▼
Pi-Pi Orchestrator (primary agent)
  │
  ├─ query_experts tool ──┬── ext-expert   (extensions API)
  │                       ├── tui-expert   (TUI components)
  │                       ├── theme-expert (color tokens)
  │                       └── config-expert (settings)
  │                       (all run simultaneously)
  │
  ▼
Orchestrator synthesizes findings → writes extension code
```

### Parallel Execution with Promise.allSettled

```typescript
const settled = await Promise.allSettled(
    queries.map(async ({ expert, question }) => {
        const result = await queryExpert(expert, question, ctx);
        return { expert, output: result.output };
    }),
);
```

`Promise.allSettled` runs all expert queries concurrently. Unlike
`Promise.all`, it does not short-circuit on failure — if one expert errors,
the others' results are still returned.

### Expert Definitions

Experts live in `.pi/agents/pi-pi/` and are read-only research agents:

```markdown
---
name: ext-expert
description: Extensions — tools, events, commands, rendering, state management
tools: read,grep,find,ls
---
You are a Pi extension expert. Research and report on the extension API...
```

### Orchestrator Template

The orchestrator's system prompt (`pi-pi/pi-orchestrator.md`) uses template
variables filled at runtime:

```typescript
systemPrompt = template
    .replace("{{EXPERT_COUNT}}", experts.size.toString())
    .replace("{{EXPERT_NAMES}}", expertNames)
    .replace("{{EXPERT_CATALOG}}", expertCatalog);
```

### Expert Dashboard Widget

A color-coded grid shows all experts with unique per-expert background colors:

```typescript
const EXPERT_COLORS: Record<string, { bg: string; br: string }> = {
    "agent-expert":  { bg: "\x1b[48;2;20;30;75m",  br: "\x1b[38;2;70;110;210m" },
    "ext-expert":    { bg: "\x1b[48;2;80;18;28m",  br: "\x1b[38;2;210;65;85m" },
    // ...
};
```

---

## Step 20: Agent Definitions and Team Configuration

### Agent Definition Format

Every agent is a Markdown file with YAML frontmatter:

```markdown
---
name: scout
description: Fast recon and codebase exploration
tools: read,grep,find,ls
---
You are a scout agent. Investigate the codebase quickly and report findings
concisely. Do NOT modify any files. Focus on structure, patterns, and key
entry points.
```

| Field | Purpose |
|-------|---------|
| `name` | Identifier used in teams.yaml and dispatch calls |
| `description` | Shown in agent catalogs and dashboards |
| `tools` | Comma-separated list of allowed tools |
| Body text | The system prompt injected for this agent |

### Agent Roles in This Project

| Agent | Tools | Role |
|-------|-------|------|
| **scout** | read, grep, find, ls | Read-only codebase exploration |
| **planner** | read, grep, find, ls | Architecture and implementation planning |
| **builder** | read, write, edit, bash, grep, find, ls | Implementation (the only agent with write access) |
| **reviewer** | read, bash, grep, find, ls | Code review and quality checks |
| **documenter** | read, write, edit, grep, find, ls | Documentation generation |
| **red-team** | read, bash, grep, find, ls | Security and adversarial testing |
| **plan-reviewer** | read, grep, find, ls | Critically evaluates implementation plans |
| **bowser** | (playwright skill) | Headless browser automation |

### Team Definitions — teams.yaml

```yaml
full:           # All agents
  - scout
  - planner
  - builder
  - reviewer
  - documenter
  - red-team

plan-build:     # Fast development cycle
  - planner
  - builder
  - reviewer

info:           # Research only
  - scout
  - documenter
  - reviewer
```

### Pipeline Definitions — agent-chain.yaml

Five predefined pipelines:

| Pipeline | Steps | Use Case |
|----------|-------|----------|
| `plan-build-review` | planner → builder → reviewer | Standard dev cycle |
| `plan-build` | planner → builder | Fast implementation |
| `scout-flow` | scout → scout → scout | Triple-pass deep recon |
| `plan-review-plan` | planner → plan-reviewer → planner | Iterative planning |
| `full-review` | scout → planner → builder → reviewer | End-to-end pipeline |

---

## Step 21: The Theme System

### Theme File Structure

Themes are JSON files in `.pi/themes/` with 51 color tokens:

```json
{
  "$schema": "https://raw.githubusercontent.com/.../theme-schema.json",
  "name": "synthwave",
  "vars": {
    "bg":      "#262335",
    "cyan":    "#36f9f6",
    "pink":    "#ff7edb",
    "green":   "#72f1b8",
    "yellow":  "#fede5d"
  },
  "colors": {
    "accent":  "cyan",
    "success": "green",
    "warning": "orange",
    "dim":     "comment",
    "muted":   "comment",
    "border":  "pink",
    "error":   "red",
    "text":    "fg"
  }
}
```

The `vars` section defines raw colors. The `colors` section maps semantic tokens
to those vars. Extensions use the semantic tokens (`theme.fg("success", text)`)
so they work with any theme.

### Color Token Conventions (from THEME.md)

| Token | Role | Examples |
|-------|------|----------|
| `success` | Primary values | Token counts, filled bars, branch name |
| `accent` | Secondary values | Percentages, tool names |
| `warning` | Punctuation/framing | Brackets, parens, pipes, cost |
| `dim` | Filler/spacing | Dashes, labels, separators |
| `muted` | Subdued text | CWD name, fallback states |

### Available Themes

catppuccin-mocha, cyberpunk, dracula, everforest, gruvbox, midnight-ocean,
nord, ocean-breeze, rose-pine, synthwave, tokyo-night

---

## Step 22: Safety Rules — damage-control-rules.yaml

The damage-control extension loads rules from `.pi/damage-control-rules.yaml`.
The file contains 100+ rules across four categories:

### bashToolPatterns

RegExp patterns matched against bash commands:

```yaml
bashToolPatterns:
  - pattern: '\brm\s+(-[^\s]*)*-[rRf]'
    reason: rm with recursive or force flags
  - pattern: '\bgit\s+push\s+.*--force(?!-with-lease)'
    reason: git push --force (use --force-with-lease)
  - pattern: '\baws\s+s3\s+rm\s+.*--recursive'
    reason: aws s3 rm --recursive (deletes all objects)
```

Some patterns have `ask: true`, which prompts for confirmation instead of
blocking outright:

```yaml
  - pattern: '\bgit\s+checkout\s+--\s*\.'
    reason: Discards all uncommitted changes
    ask: true
```

### Path-Based Rules

```yaml
zeroAccessPaths:       # Cannot read, write, or access
  - ".env"
  - "~/.ssh/"
  - "*.pem"

readOnlyPaths:         # Can read, cannot modify
  - /etc/
  - node_modules/
  - "*.lock"

noDeletePaths:         # Cannot delete or move
  - .git/
  - LICENSE
  - README.md
```

---

## Step 23: How Extensions Compose

Extensions are designed to stack. The `justfile` shows the standard combinations:

```bash
# Minimal footer + theme cycling
just ext-minimal   →  pi -e extensions/minimal.ts -e extensions/theme-cycler.ts

# Subagent widgets + pure-focus footer + theme cycling
just ext-subagent-widget  →  pi -e extensions/subagent-widget.ts \
                                 -e extensions/pure-focus.ts \
                                 -e extensions/theme-cycler.ts
```

### Composition Rules

1. **Theme:** The first `-e` extension controls the theme (via `primaryExtensionName()`)
2. **Footer:** The last extension to call `setFooter()` wins
3. **Widgets:** Each widget has a unique ID; extensions can coexist in the widget area
4. **Commands:** Each command name must be unique across all loaded extensions
5. **Shortcuts:** Each key combo must be unique
6. **Events:** All extensions' event handlers fire in load order; blocking (`tool_call`)
   applies from any handler
7. **Tools:** All registered tools are available to the AI simultaneously

### Common Stack Patterns

| Pattern | Extensions | Purpose |
|---------|------------|---------|
| Dev focus | `minimal` + `theme-cycler` | Compact UI with theme control |
| Task discipline | `tilldone` + `theme-cycler` | Forced planning with themes |
| Safety net | `damage-control` + `minimal` + `theme-cycler` | Safe operations |
| Orchestration | `agent-team` + `theme-cycler` | Multi-agent with themes |
| Meta-agent | `pi-pi` + `theme-cycler` | Extension building with themes |

---

## Step 24: Patterns Cheat Sheet

A quick reference for the core patterns used across extensions.

### Extension Skeleton

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";

export default function (pi: ExtensionAPI) {
    // Register tools, commands, shortcuts at top level

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        // Setup: footers, widgets, status, state reconstruction
    });
}
```

### Event Hooks

| Event | When It Fires | Common Use |
|-------|---------------|------------|
| `session_start` | New session begins | Setup UI, load config |
| `session_switch` | User switches branches | Reconstruct state |
| `session_fork` | Session forks | Reconstruct state |
| `session_shutdown` | Session ending | Cleanup timers |
| `input` | User submits message | Gate/block input |
| `before_agent_start` | Before AI turn | Modify system prompt |
| `tool_call` | AI wants to call a tool | Block/allow/confirm |
| `tool_execution_end` | Tool finished | Count, log, track |
| `agent_end` | AI turn completed | Nudge, summarize |

### UI APIs

| API | Purpose |
|-----|---------|
| `ctx.ui.setFooter(factory)` | Replace footer (bottom of screen) |
| `ctx.ui.setWidget(id, factory)` | Add/update/remove named widget |
| `ctx.ui.setStatus(key, text)` | Update status line |
| `ctx.ui.notify(msg, level)` | Show notification |
| `ctx.ui.select(title, items)` | Selection dialog |
| `ctx.ui.input(title, placeholder)` | Text input dialog |
| `ctx.ui.confirm(title, msg)` | Yes/no confirmation |
| `ctx.ui.custom(factory, options)` | Full-screen custom UI |
| `ctx.ui.setTheme(name)` | Switch theme |
| `ctx.ui.setTitle(text)` | Set terminal title |

### Tool Registration

```typescript
pi.registerTool({
    name: "my_tool",
    description: "What the AI sees",
    parameters: Type.Object({ arg: Type.String() }),
    execute: async (callId, args, signal, onUpdate, ctx) => {
        return { content: [{ type: "text", text: "result" }] };
    },
});
```

### Blocking Gate Pattern

```typescript
pi.on("tool_call", async (event) => {
    if (shouldBlock) {
        return { block: true, reason: "Explain why" };
    }
    return { block: false };
});
```

### System Prompt Injection

```typescript
pi.on("before_agent_start", async (event) => {
    return {
        systemPrompt: event.systemPrompt + "\n\nAdditional instructions",
    };
});
```

### Subagent Spawning

```typescript
const proc = spawn("pi", [
    "--mode", "json", "-p",
    "--no-extensions", "--model", model,
    "--tools", "read,bash,grep,find,ls",
    prompt,
]);
// Parse JSONL from proc.stdout
// Deliver results via pi.sendMessage({}, { triggerTurn: true })
```

### State Reconstruction

```typescript
for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.role === "toolResult") {
        const details = entry.message.details;
        // Rebuild state from stored snapshots
    }
}
```

---

## Next Steps

Now that you understand the full codebase, here are paths forward:

1. **Run every extension** — work through `just ext-*` commands sequentially
2. **Modify an extension** — change the context bar style in `minimal.ts`
3. **Build your own** — follow the tutorials in `docs/tutorials/` (10 guided builds)
4. **Read the architecture** — `docs/ARCHITECTURE.md` has flow diagrams and deeper analysis
5. **Study the API** — `docs/DEVELOPER_GUIDE.md` is a full API reference
6. **Follow the study plan** — `docs/STUDY_PLAN.md` is a 9-phase mastery guide
