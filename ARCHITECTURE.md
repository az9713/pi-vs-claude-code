# Pi vs Claude Code — Architecture Document

## Table of Contents

1. [Overview and Purpose](#1-overview-and-purpose)
2. [Technology Stack](#2-technology-stack)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Project File Structure](#4-project-file-structure)
5. [Extension System Architecture](#5-extension-system-architecture)
6. [Extension Catalog](#6-extension-catalog)
7. [Event System](#7-event-system)
8. [UI Component Architecture](#8-ui-component-architecture)
9. [Multi-Agent Orchestration](#9-multi-agent-orchestration)
10. [Safety and Access Control Architecture](#10-safety-and-access-control-architecture)
11. [Theme System Architecture](#11-theme-system-architecture)
12. [Agent Definition Format](#12-agent-definition-format)
13. [Session and State Management](#13-session-and-state-management)
14. [Communication Flow Diagrams](#14-communication-flow-diagrams)
15. [Data Flow: How a User Prompt Becomes an Action](#15-data-flow-how-a-user-prompt-becomes-an-action)
16. [Cross-Agent Integration](#16-cross-agent-integration)
17. [Key Code Patterns and Idioms](#17-key-code-patterns-and-idioms)

---

## 1. Overview and Purpose

This repository is a **playground of 16 Pi Coding Agent extensions**. Pi is an open-source, MIT-licensed terminal-based AI coding assistant — comparable to Claude Code, but designed from the ground up to be extensible at the source level.

The project serves three goals:

1. **Showcase extensibility.** Every aspect of Pi — its UI, its tool set, its safety behavior, its agent identity — can be overridden from a single TypeScript file.
2. **Demonstrate multi-agent orchestration.** Four distinct patterns show how separate Pi processes can collaborate: dispatcher, pipeline, parallel research, and background subagent.
3. **Provide a safety reference.** The `damage-control` extension demonstrates production-grade input validation for AI agents through a declarative YAML rule set.

Pi vs Claude Code is not a competitive benchmark. It is a demonstration that an open-source tool can match and in some respects exceed the customizability of proprietary alternatives.

---

## 2. Technology Stack

This section explains each tool for developers who know C, C++, or Java but are new to the TypeScript and web tooling ecosystem.

### TypeScript

TypeScript is JavaScript with a static type system added on top. Unlike Java, there is no separate compilation step that produces `.class` files — TypeScript is transpiled to JavaScript at runtime or build time. In this project, Pi uses **jiti** (Just-In-Time TypeScript) to load `.ts` extension files directly without a separate build step. Think of jiti as a JIT compiler analogous to the JVM's HotSpot, but for TypeScript.

### Bun

Bun is a JavaScript runtime and package manager. It serves the same role as the JVM for running JavaScript code, and the same role as Maven or Gradle for managing dependencies. Where Maven downloads JARs to `~/.m2`, Bun downloads packages to `node_modules/`. The `bun install` command reads `package.json` (analogous to `pom.xml` or `build.gradle`) and installs all declared dependencies.

### just

`just` is a command runner. It reads a `justfile` in the same way that `make` reads a `Makefile`. Each recipe in the justfile is a named shortcut for a shell command. For example, `just ext-minimal` expands to `pi -e extensions/minimal.ts -e extensions/theme-cycler.ts`. This removes the need to remember long command strings.

### Node.js child_process

Several multi-agent extensions use Node.js's built-in `child_process.spawn()` to launch separate `pi` CLI processes. This is analogous to `Runtime.exec()` in Java or `fork()`/`exec()` in C. The parent process reads JSON events from the child's stdout line by line.

### TypeBox

TypeBox is a library for defining JSON schema objects that also produce TypeScript types. It is used to declare tool parameter schemas. The Pi framework reads these schemas to validate tool inputs and generate documentation. TypeBox's `Type.Object()`, `Type.String()`, and `Type.Number()` are roughly analogous to defining a POJO with validation annotations in Java Bean Validation.

### YAML

YAML is a human-readable data serialization format used here for configuration files: `teams.yaml` (team definitions), `agent-chain.yaml` (pipeline definitions), and `damage-control-rules.yaml` (safety rules). It is analogous to XML or JSON but with less syntactic noise.

---

## 3. High-Level Architecture

Pi is structured as four npm packages. Extensions in this project are consumers of the top-most package, `@mariozechner/pi-coding-agent`.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        pi-vs-claude-code                            │
│                   (This Repository: extensions/)                    │
│                                                                     │
│  Each extension is a TypeScript file that calls into the            │
│  ExtensionAPI (pi) and ExtensionContext (ctx) interfaces.           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ imports
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              @mariozechner/pi-coding-agent                          │
│                   (CLI + Extension API)                             │
│                                                                     │
│  • Entry point for the `pi` CLI command                             │
│  • ExtensionAPI: registerTool, registerCommand, registerShortcut,   │
│    registerFlag, on(event), sendMessage, sendUserMessage, exec,     │
│    setActiveTools, getActiveTools, getAllTools, setModel,           │
│    getThinkingLevel, setThinkingLevel, appendEntry                  │
│  • ExtensionContext: cwd, model, hasUI, ui, sessionManager,         │
│    getContextUsage, getSystemPrompt, abort                          │
│  • Built-in tools: read, write, edit, bash, grep, find, ls         │
│  • Theme system: Theme.fg(), Theme.bg(), Theme.bold()              │
│  • DynamicBorder, getMarkdownTheme utilities                        │
└───────────────┬────────────────────────────┬────────────────────────┘
                │ depends on                  │ depends on
                ▼                             ▼
┌───────────────────────────┐   ┌─────────────────────────────────────┐
│  @mariozechner/pi-tui     │   │  @mariozechner/pi-agent-core        │
│  (Terminal UI components) │   │  (Agent loop + session management)  │
│                           │   │                                     │
│  • Text: single-line or   │   │  • Tool execution engine            │
│    multi-line rendered    │   │  • Session branching (fork/switch)  │
│    text block             │   │  • Event bus (25+ event types)      │
│  • Box: padded container  │   │  • Message history management       │
│  • Container: vertical    │   │  • Context window tracking          │
│    stack of children      │   │  • Turn lifecycle management        │
│  • Markdown: rendered     │   │                                     │
│    Markdown block         │   └─────────────────────────────────────┘
│  • Spacer: blank lines    │
│  • matchesKey, Key enum   │
│  • truncateToWidth        │
│  • visibleWidth           │
│  • wrapTextWithAnsi       │
└───────────────────────────┘
                │ depends on
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    @mariozechner/pi-ai                              │
│              (LLM abstraction + model registry)                     │
│                                                                     │
│  • 324 models across 20+ providers                                  │
│  • Provider adapters: OpenAI, Anthropic, Google, OpenRouter, etc.   │
│  • Unified message format across all providers                      │
│  • Usage tracking: input tokens, output tokens, cost                │
│  • StringEnum helper (required for Google Gemini compatibility)     │
│  • AssistantMessage type with usage.input, usage.output, usage.cost │
└─────────────────────────────────────────────────────────────────────┘
```

### Dependency Direction

Dependencies only flow downward. Extensions cannot import from `pi-agent-core` directly — they interact with the agent loop exclusively through the `ExtensionAPI` and `ExtensionContext` interfaces provided by `pi-coding-agent`. This enforces a clean boundary: extensions cannot accidentally break internal agent state.

---

## 4. Project File Structure

```
pi-vs-claude-code/
│
├── extensions/                    # All 16 extension source files (.ts)
│   ├── pure-focus.ts              # Strips footer and status line entirely
│   ├── minimal.ts                 # Model name + 10-block context meter
│   ├── tool-counter.ts            # 2-line footer: tokens, cost, branch, tally
│   ├── tool-counter-widget.ts     # Per-tool counts in a colored widget
│   ├── purpose-gate.ts            # Blocks input until session intent declared
│   ├── cross-agent.ts             # Scans .claude/.gemini/.codex for assets
│   ├── session-replay.ts          # Scrollable timeline overlay
│   ├── theme-cycler.ts            # Ctrl+X/Ctrl+Q cycling, /theme command
│   ├── system-select.ts           # /system command to switch agent personas
│   ├── damage-control.ts          # Real-time safety auditing from YAML rules
│   ├── tilldone.ts                # 3-state task lifecycle (idle/inprog/done)
│   ├── subagent-widget.ts         # /sub spawns background Pi subagents
│   ├── agent-team.ts              # Dispatcher orchestrator + grid dashboard
│   ├── agent-chain.ts             # Sequential pipeline orchestrator
│   ├── pi-pi.ts                   # Meta-agent with parallel expert research
│   └── themeMap.ts                # Central theme-to-extension mapping helper
│
├── .pi/                           # Pi workspace configuration
│   ├── settings.json              # Pi workspace settings (provider, model)
│   ├── damage-control-rules.yaml  # 146 safety rules across 4 categories
│   │
│   ├── agents/                    # Agent definition files
│   │   ├── planner.md             # read/grep/find/ls only — no writes
│   │   ├── builder.md             # Full tool access: read/write/edit/bash
│   │   ├── scout.md               # Read-only reconnaissance
│   │   ├── reviewer.md            # read/bash — code review persona
│   │   ├── documenter.md          # Documentation specialist
│   │   ├── red-team.md            # Security/adversarial reviewer
│   │   ├── plan-reviewer.md       # Plan critique specialist
│   │   ├── bowser.md              # Frontend/browser testing specialist
│   │   ├── teams.yaml             # Team definitions (full/plan-build/info/...)
│   │   ├── agent-chain.yaml       # Pipeline definitions (5 named chains)
│   │   │
│   │   └── pi-pi/                 # Expert agents for pi-pi meta-agent
│   │       ├── pi-orchestrator.md # Orchestrator system prompt template
│   │       ├── ext-expert.md      # Extensions API expert
│   │       ├── theme-expert.md    # Theme JSON format expert
│   │       ├── tui-expert.md      # Terminal UI components expert
│   │       ├── skill-expert.md    # Skills (SKILL.md packages) expert
│   │       ├── config-expert.md   # settings.json + providers expert
│   │       ├── prompt-expert.md   # Prompt template expert
│   │       ├── agent-expert.md    # Agent definition format expert
│   │       ├── keybinding-expert.md  # Keyboard shortcut expert
│   │       └── cli-expert.md      # CLI flags and invocation expert
│   │
│   ├── skills/                    # Custom Pi skills
│   │   └── bowser.md              # Browser testing skill
│   │
│   ├── themes/                    # 11 custom theme JSON files
│   │   ├── synthwave.json         # Neon purple/cyan/pink
│   │   ├── catppuccin-mocha.json  # Soft pastel dark
│   │   ├── dracula.json           # Classic purple dark
│   │   ├── tokyo-night.json       # Dark blue/purple
│   │   ├── cyberpunk.json         # High-contrast neon
│   │   ├── nord.json              # Arctic blue-grey
│   │   ├── gruvbox.json           # Warm earthy brown/orange
│   │   ├── everforest.json        # Muted green/teal
│   │   ├── rose-pine.json         # Soft warm purple
│   │   ├── midnight-ocean.json    # Deep blue-teal
│   │   └── ocean-breeze.json      # Light cool-toned blue
│   │
│   └── agent-sessions/            # Ephemeral subprocess session files
│                                  # (gitignored — created at runtime)
│
├── .claude/
│   └── commands/
│       └── prime.md               # Claude Code /prime slash command
│
├── specs/                         # Feature specifications
│   ├── agent-forge.md             # Spec for agent construction tooling
│   ├── agent-workflow.md          # Multi-agent workflow spec
│   ├── damage-control.md          # Safety auditing spec
│   └── pi-pi.md                   # Pi-pi meta-agent spec
│
├── justfile                       # 20+ task runner recipes
├── package.json                   # Dependencies (yaml ^2.8.0)
├── bun.lock                       # Locked dependency versions
├── CLAUDE.md                      # Agent conventions (for Claude Code)
├── COMPARISON.md                  # CC vs Pi feature comparison
├── RESERVED_KEYS.md               # Keyboard shortcut reference
├── THEME.md                       # Color token conventions
├── TOOLS.md                       # Built-in tool signatures
└── .env.sample                    # API key template
```

---

## 5. Extension System Architecture

### How Extensions Load

When you run `pi -e extensions/minimal.ts`, the Pi CLI:

1. Resolves the path to the `.ts` file.
2. Uses **jiti** to load and execute the file as a TypeScript module without a prior compilation step.
3. Calls the file's **default export function**, passing it an `ExtensionAPI` object.
4. The extension registers event listeners, tools, commands, and shortcuts during this synchronous initialization phase.
5. Pi then enters its normal interactive loop, calling the registered handlers as events occur.

Multiple extensions can be stacked with multiple `-e` flags:

```
pi -e extensions/damage-control.ts -e extensions/minimal.ts -e extensions/theme-cycler.ts
```

All three default-export functions run in sequence during initialization. Each extension's handlers are added to the same event bus, so they all receive the same events.

### Extension Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                   Extension Initialization                       │
│                                                                  │
│  pi -e extensions/myext.ts                                       │
│       │                                                          │
│       ▼                                                          │
│  jiti loads .ts file                                             │
│       │                                                          │
│       ▼                                                          │
│  default export function(pi: ExtensionAPI) {                     │
│      pi.registerTool(...)        ← registered immediately        │
│      pi.registerCommand(...)     ← registered immediately        │
│      pi.registerShortcut(...)    ← registered immediately        │
│      pi.on("session_start", ...)  ← listener queued             │
│      pi.on("tool_call", ...)      ← listener queued             │
│      pi.on("agent_end", ...)      ← listener queued             │
│  }                                                               │
│       │                                                          │
│       ▼                                                          │
│  Pi enters interactive loop                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   Runtime Event Flow                             │
│                                                                  │
│  User starts session                                             │
│       │                                                          │
│       ▼                                                          │
│  "session_start" fires → extension calls ctx.ui.setFooter(),    │
│                          ctx.ui.setTheme(), ctx.ui.setStatus()   │
│       │                                                          │
│       ▼                                                          │
│  User types prompt → "input" fires → extension can block        │
│       │               (return { action: "handled" })             │
│       ▼                                                          │
│  "before_agent_start" fires → extension can inject systemPrompt │
│       │                                                          │
│       ▼                                                          │
│  LLM decides to call a tool → "tool_call" fires →               │
│                               extension can block tool           │
│                               (return { block: true, reason })  │
│       │                                                          │
│       ▼                                                          │
│  Tool executes → "tool_execution_start/update/end" fires         │
│       │                                                          │
│       ▼                                                          │
│  LLM responds → "message_start/update/end" fires                │
│       │                                                          │
│       ▼                                                          │
│  Agent finishes turn → "agent_end" fires →                       │
│                        extension can inject follow-up message   │
└─────────────────────────────────────────────────────────────────┘
```

### Extension API Surface

**ExtensionAPI (the `pi` parameter)**

| Method | Description |
|---|---|
| `registerTool(def)` | Add a new tool the LLM can call |
| `registerCommand(name, def)` | Add a `/name` slash command |
| `registerShortcut(key, def)` | Add a keyboard shortcut (e.g. `"ctrl+x"`) |
| `registerFlag(name, def)` | Add a CLI flag |
| `on(event, handler)` | Subscribe to an event type |
| `sendMessage(msg, opts)` | Inject a message into the session |
| `sendUserMessage(text)` | Inject a user-role message |
| `exec(cmd)` | Run a shell command from extension code |
| `setActiveTools(names)` | Restrict the active tool set |
| `getActiveTools()` | Get current active tool names |
| `getAllTools()` | Get all registered tool names |
| `setModel(modelId)` | Switch the LLM model |
| `getThinkingLevel()` | Get current thinking budget |
| `setThinkingLevel(level)` | Set thinking budget |
| `appendEntry(type, data)` | Append a custom entry to session history |

**ExtensionContext (the `ctx` parameter, passed to event handlers)**

| Property/Method | Description |
|---|---|
| `cwd` | Current working directory |
| `model` | Current model info (id, provider, contextWindow) |
| `hasUI` | Whether the session is running in interactive mode |
| `ui` | UI API (see section 8) |
| `sessionManager.getBranch()` | Get ordered message history for current branch |
| `getContextUsage()` | Get token usage as `{ used, total, percent }` |
| `getSystemPrompt()` | Get the current system prompt text |
| `abort()` | Abort the current agent turn |

---

## 6. Extension Catalog

| # | Extension | File | Category | Key Feature |
|---|---|---|---|---|
| 1 | pure-focus | `pure-focus.ts` | UI | Returns empty footer array, removing all status chrome |
| 2 | minimal | `minimal.ts` | UI | Single-line footer: model name + `[###-------] 30%` context bar |
| 3 | tool-counter | `tool-counter.ts` | UI | 2-line footer: tokens in/out + cost (line 1), cwd/branch + tool tally (line 2) |
| 4 | tool-counter-widget | `tool-counter-widget.ts` | UI | Above-editor widget showing colored per-tool call counts |
| 5 | purpose-gate | `purpose-gate.ts` | Discipline | Blocks all input until user types a session intent; injects purpose into system prompt |
| 6 | tilldone | `tilldone.ts` | Discipline | 3-state task lifecycle; blocks tools unless a task is in-progress |
| 7 | session-replay | `session-replay.ts` | UI | Scrollable timeline overlay with up/down navigation and Enter-to-expand |
| 8 | theme-cycler | `theme-cycler.ts` | UI | Ctrl+X forward / Ctrl+Q backward; /theme picker; color swatch widget auto-dismiss |
| 9 | system-select | `system-select.ts` | Configuration | /system dialog scans all agent directories and switches system prompt + tool set |
| 10 | cross-agent | `cross-agent.ts` | Integration | Scans .claude/.gemini/.codex for commands/skills/agents; registers commands in Pi |
| 11 | damage-control | `damage-control.ts` | Safety | Intercepts tool_call events; evaluates against 146 YAML rules; blocks or prompts |
| 12 | subagent-widget | `subagent-widget.ts` | Orchestration | /sub spawns background Pi subprocess; live widget per agent; /subcont to resume |
| 13 | agent-team | `agent-team.ts` | Orchestration | Dispatcher: primary agent has only dispatch_agent tool; routes to specialist subprocesses |
| 14 | agent-chain | `agent-chain.ts` | Orchestration | Pipeline: run_chain tool executes sequential steps; $INPUT/$ORIGINAL variable substitution |
| 15 | pi-pi | `pi-pi.ts` | Orchestration | Meta-agent: query_experts tool fans out to parallel specialist subprocesses |
| 16 | themeMap | `themeMap.ts` | Utility | Not an extension; shared helper providing `applyExtensionDefaults(import.meta.url, ctx)` |

---

## 7. Event System

### Event Categories and Types

```
┌──────────────────────────────────────────────────────────────┐
│                    Pi Event System                            │
│                                                              │
│  ┌─ Session ──────────────────────────────────────────────┐  │
│  │  session_start        fired when Pi session begins     │  │
│  │  session_shutdown     fired when Pi session ends       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Input ────────────────────────────────────────────────┐  │
│  │  input                fired on every user prompt       │  │
│  │                       can return { action: "handled" } │  │
│  │                       to swallow the input             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Agent / Turn ─────────────────────────────────────────┐  │
│  │  before_agent_start   can inject/override systemPrompt │  │
│  │  agent_start          agent loop begins                │  │
│  │  agent_end            agent loop ends (all turns done) │  │
│  │  turn_start           one LLM request begins           │  │
│  │  turn_end             one LLM request ends             │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Tool ─────────────────────────────────────────────────┐  │
│  │  tool_call            LLM requests a tool call         │  │
│  │                       can return { block: true, reason}│  │
│  │  tool_result          tool call returned a result      │  │
│  │  tool_execution_start tool begins executing            │  │
│  │  tool_execution_update partial result available        │  │
│  │  tool_execution_end   tool finished executing          │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Message ──────────────────────────────────────────────┐  │
│  │  message_start        LLM begins a response            │  │
│  │  message_update       LLM text delta or tool call chunk│  │
│  │  message_end          LLM response complete            │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Session Branching ────────────────────────────────────┐  │
│  │  session_before_fork  about to fork session branch     │  │
│  │  session_fork         branch forked                    │  │
│  │  session_before_switch about to switch branch          │  │
│  │  session_switch       branch switched                  │  │
│  │  session_before_tree  about to show branch tree        │  │
│  │  session_tree         branch tree displayed            │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─ Model / Context ──────────────────────────────────────┐  │
│  │  model_select         user changed the active model    │  │
│  │  context              context window usage updated     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Event Reference Table

| Event | Handler Signature | Return Value | Common Use |
|---|---|---|---|
| `session_start` | `(event, ctx) => Promise<void>` | none | Initialize UI, load state, set theme |
| `session_shutdown` | `(event) => Promise<void>` | none | Clean up timers and resources |
| `input` | `(event, ctx) => Promise<{action}>` | `{action: "continue"\|"handled"}` | Gate user input (purpose-gate) |
| `before_agent_start` | `(event, ctx) => Promise<{systemPrompt?}>` | optional `{systemPrompt}` | Inject or replace system prompt |
| `agent_start` | `(event, ctx) => Promise<void>` | none | Track agent start time |
| `agent_end` | `(event, ctx) => Promise<void>` | none | Auto-nudge (tilldone), cleanup |
| `turn_start` | `(event, ctx) => Promise<void>` | none | Track per-turn state |
| `turn_end` | `(event, ctx) => Promise<void>` | none | Accumulate turn stats |
| `tool_call` | `(event, ctx) => Promise<{block, reason?}>` | `{block: boolean, reason?}` | Block dangerous tool calls |
| `tool_result` | `(event, ctx) => Promise<void>` | none | Observe tool output |
| `tool_execution_start` | `(event, ctx) => Promise<void>` | none | Count tool executions |
| `tool_execution_update` | `(event, ctx) => Promise<void>` | none | Stream partial results |
| `tool_execution_end` | `(event, ctx) => Promise<void>` | none | Tally completed executions |
| `message_start` | `(event, ctx) => Promise<void>` | none | Track message timing |
| `message_update` | `(event, ctx) => Promise<void>` | none | Stream text deltas to widgets |
| `message_end` | `(event, ctx) => Promise<void>` | none | Capture usage/cost data |
| `session_fork` | `(event, ctx) => Promise<void>` | none | Reconstruct state after fork |
| `session_switch` | `(event, ctx) => Promise<void>` | none | Reconstruct state after switch |
| `session_tree` | `(event, ctx) => Promise<void>` | none | Reconstruct state after tree |
| `model_select` | `(event, ctx) => Promise<void>` | none | React to model changes |
| `context` | `(event, ctx) => Promise<void>` | none | React to context window changes |

### Type-Safe Tool Call Narrowing

The `isToolCallEventType()` helper narrows the generic `tool_call` event to a specific tool's typed input. This is the pattern used in `damage-control.ts`:

```typescript
// Without narrowing: event.input is unknown
pi.on("tool_call", async (event, ctx) => {
    // With narrowing: event.input.path is string, event.input.content is string
    if (isToolCallEventType("read", event)) {
        console.log(event.input.path);
    }
    if (isToolCallEventType("bash", event)) {
        console.log(event.input.command);
    }
});
```

---

## 8. UI Component Architecture

The terminal UI is built from composable components from `@mariozechner/pi-tui`. All components implement a `render(width: number): string[]` method that returns an array of ANSI-colored strings, one per terminal line.

### Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Terminal Window                                  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                      Widget Zone                               │  │
│  │  (belowEditor placement — stacks multiple named widgets)       │  │
│  │                                                                │  │
│  │  ┌─ tilldone-current ─────────────────────────────────────┐   │  │
│  │  │  ● WORKING ON  #3  Implement authentication module     │   │  │
│  │  └────────────────────────────────────────────────────────┘   │  │
│  │  ┌─ tool-counter ─────────────────────────────────────────┐   │  │
│  │  │  Tools (12):  [bash 3]  [read 7]  [write 2]            │   │  │
│  │  └────────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   Conversation / Chat Area                     │  │
│  │   User: implement authentication                               │  │
│  │   Assistant: I'll start by creating the auth module...         │  │
│  │   [Tool: read src/auth.ts]                                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                         Editor / Input                        │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    Footer Zone (custom)                        │  │
│  │  gemini-3-flash    [###-------] 30%       12k in  2k out $0.01│  │
│  │  src/auth  (main)                  read 7  bash 3  write 2    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   Status Bar (one-liner)                       │  │
│  │  📋 TillDone: 5 tasks (3 remaining)                            │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Dialog / Overlay Types

Extensions can create blocking dialogs through `ctx.ui`:

| Method | Description | Blocks input? |
|---|---|---|
| `ctx.ui.notify(msg, type?)` | Toast notification. Types: `"info"`, `"success"`, `"warning"`, `"error"` | No |
| `ctx.ui.input(title, placeholder)` | Text input dialog. Returns the entered string. | Yes (awaited) |
| `ctx.ui.confirm(title, body, opts?)` | Yes/No confirmation dialog. Returns boolean. | Yes (awaited) |
| `ctx.ui.select(title, options)` | Scrollable list picker. Returns selected string. | Yes (awaited) |
| `ctx.ui.custom(factory, opts?)` | Full custom overlay with `render()` and `handleInput()`. | Yes (awaited) |
| `ctx.ui.editor(initial?)` | Open a text editor. Returns edited string. | Yes (awaited) |

### UI Component Types (from @mariozechner/pi-tui)

| Component | Constructor | Purpose |
|---|---|---|
| `Text` | `new Text(content, paddingTop, paddingLeft)` | Renders a string. Handles multiline. |
| `Box` | `new Box(paddingTop, paddingLeft, styleFn?)` | Container with optional background style |
| `Container` | `new Container()` | Vertical stack; `addChild()` adds components |
| `Markdown` | `new Markdown(text, pTop, pLeft, mdTheme)` | Rendered Markdown with syntax highlighting |
| `Spacer` | `new Spacer(lines)` | N blank lines |
| `DynamicBorder` | `new DynamicBorder(styleFn)` | Full-width horizontal line with applied style |

### Theme Color Tokens

The `Theme` object (passed to footer/widget render factories) provides 51 semantic color tokens. The key ones:

| Token | Semantic Role |
|---|---|
| `accent` | Primary highlight color (e.g. cyan in synthwave) |
| `success` | Positive state (e.g. green) |
| `warning` | Caution state (e.g. orange) |
| `error` | Failure state (e.g. red) |
| `dim` | Reduced-emphasis text |
| `muted` | Secondary text |
| `border` | Box borders |
| `borderMuted` | Subtle borders |
| `toolTitle` | Tool name in tool call display |
| `selectedBg` | Background for selected items |
| `userMessageBg` | User message background |
| `mdHeading` | Markdown heading color |
| `syntaxKeyword` | Syntax-highlighted keyword color |

---

## 9. Multi-Agent Orchestration

All multi-agent extensions share a common technical mechanism: they use Node.js `child_process.spawn()` to launch separate `pi` CLI processes with `--mode json` (which causes Pi to emit JSON events to stdout) and `-p` (print mode — non-interactive). The parent reads events line-by-line from the child's stdout.

### Subprocess Spawn Pattern (common to all 4 orchestration styles)

```typescript
const proc = spawn("pi", [
    "--mode", "json",      // emit JSON events to stdout
    "-p",                  // non-interactive print mode
    "--no-extensions",     // do not load any extensions
    "--model", model,      // e.g. "openrouter/google/gemini-3-flash-preview"
    "--tools", toolList,   // e.g. "read,grep,find,ls"
    "--thinking", "off",
    "--session", sessionFile,  // path to .json session file for resumption
    prompt,                // the task as a positional argument
], {
    stdio: ["ignore", "pipe", "pipe"],
    env: { ...process.env },
});

// Read JSON events line by line
proc.stdout.on("data", (chunk) => {
    buffer += chunk;
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";
    for (const line of lines) {
        const event = JSON.parse(line);
        if (event.type === "message_update") { /* stream text */ }
        if (event.type === "tool_execution_start") { /* count tools */ }
        if (event.type === "message_end") { /* capture usage */ }
        if (event.type === "agent_end") { /* capture final result */ }
    }
});
```

---

### Pattern 1: Dispatcher Pattern (agent-team.ts)

The primary agent has its tool set locked to a single tool: `dispatch_agent`. It cannot read files, write code, or run bash commands. It can only delegate.

```
┌──────────────────────────────────────────────────────────────────┐
│                  agent-team.ts — Dispatcher Pattern              │
│                                                                  │
│  User Prompt                                                     │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │             Primary Pi Agent (Dispatcher)               │     │
│  │                                                         │     │
│  │  tools: [dispatch_agent]  ← ONLY tool available         │     │
│  │  system prompt: dynamically built from agent catalog    │     │
│  │                                                         │     │
│  │  The LLM must analyze the user request and decide       │     │
│  │  which specialist(s) to call and in what order.         │     │
│  └────────────────────────────┬────────────────────────────┘     │
│                               │ calls dispatch_agent()           │
│             ┌─────────────────┼──────────────────┐               │
│             │                 │                  │               │
│             ▼                 ▼                  ▼               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐          │
│  │ scout agent  │   │planner agent │   │builder agent │          │
│  │              │   │              │   │              │          │
│  │ pi subprocess│   │ pi subprocess│   │ pi subprocess│          │
│  │ --no-ext     │   │ --no-ext     │   │ --no-ext     │          │
│  │ tools:       │   │ tools:       │   │ tools:       │          │
│  │ read,grep,   │   │ read,grep,   │   │ read,write,  │          │
│  │ find,ls      │   │ find,ls      │   │ edit,bash,   │          │
│  │              │   │              │   │ grep,find,ls │          │
│  │ session:     │   │ session:     │   │ session:     │          │
│  │ scout.json   │   │ planner.json │   │ builder.json │          │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘          │
│         │ output            │ output            │ output           │
│         └─────────────────── ───────────────────┘                 │
│                               │                                   │
│                               ▼                                   │
│              Dispatcher synthesizes results for user              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │                Grid Dashboard Widget                    │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │     │
│  │  │  Scout   │  │ Planner  │  │ Builder  │              │     │
│  │  │ ● running│  │ ○ idle   │  │ ✓ done   │              │     │
│  │  │ [##---]  │  │ [-----]  │  │ [#####]  │              │     │
│  │  │ "scan..."│  │ "plan"   │  │ "write.."│              │     │
│  │  └──────────┘  └──────────┘  └──────────┘              │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Sessions are persistent within a Pi session.                    │
│  /agents-team to switch teams.                                   │
│  Agent session files wiped on session_start.                     │
└──────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- `pi.setActiveTools(["dispatch_agent"])` is called in `session_start`, stripping all built-in tools from the primary agent.
- Each specialist's session file is stored in `.pi/agent-sessions/<name>.json`. On the first dispatch the file does not exist (fresh start). On subsequent dispatches the same session file is reused (`-c` flag), giving the specialist memory of previous work.
- Agent definitions are loaded from `agents/*.md`, `.claude/agents/*.md`, and `.pi/agents/*.md` — any of the three directories can supply agents.

---

### Pattern 2: Pipeline Pattern (agent-chain.ts)

The pipeline runs agents sequentially. The output of each step becomes the input to the next via `$INPUT` variable substitution. The primary agent retains its own tools and can decide whether to invoke the chain or handle a request directly.

```
┌──────────────────────────────────────────────────────────────────┐
│                agent-chain.ts — Pipeline Pattern                 │
│                                                                  │
│  User Prompt                                                     │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │         Primary Pi Agent (has ALL tools + run_chain)    │     │
│  │                                                         │     │
│  │  Decides: is this a simple lookup or real work?         │     │
│  │  If real work → calls run_chain(task)                   │     │
│  │  If simple   → handles directly with built-in tools     │     │
│  └────────────────────────────┬────────────────────────────┘     │
│                               │ calls run_chain()                │
│                               ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │             Sequential Pipeline Executor                │     │
│  │                                                         │     │
│  │  Step 1: planner                                        │     │
│  │      prompt: "Plan the implementation for: $INPUT"      │     │
│  │          │                                              │     │
│  │          │ spawn pi subprocess for planner              │     │
│  │          │ wait for completion                          │     │
│  │          │ capture output                               │     │
│  │          ▼                                              │     │
│  │  Step 2: builder                                        │     │
│  │      prompt: "Implement this plan:\n\n$INPUT"           │     │
│  │      ($INPUT = planner output)                          │     │
│  │          │                                              │     │
│  │          │ spawn pi subprocess for builder              │     │
│  │          │ wait for completion                          │     │
│  │          │ capture output                               │     │
│  │          ▼                                              │     │
│  │  Step 3: reviewer                                       │     │
│  │      prompt: "Review this implementation:\n\n$INPUT"    │     │
│  │      ($INPUT = builder output)                          │     │
│  │          │                                              │     │
│  │          ▼                                              │     │
│  │  Final output returned to primary agent                 │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Variable substitution:                                          │
│    $INPUT    = previous step's output (or user prompt for step 1)│
│    $ORIGINAL = the user's original prompt (all steps)            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │             Pipeline Progress Widget                    │     │
│  │  ┌──────────┐     ┌──────────┐     ┌──────────┐        │     │
│  │  │ Planner  │──▶  │ Builder  │──▶  │ Reviewer │        │     │
│  │  │ ✓ done   │     │ ● running│     │ ○ pending│        │     │
│  │  │ 12s      │     │ 34s      │     │          │        │     │
│  │  └──────────┘     └──────────┘     └──────────┘        │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

**Available chains (from agent-chain.yaml):**

| Chain | Steps | Description |
|---|---|---|
| `plan-build-review` | planner → builder → reviewer | Standard development cycle |
| `plan-build` | planner → builder | Fast two-step without review |
| `scout-flow` | scout → scout → scout | Triple-validation reconnaissance |
| `plan-review-plan` | planner → plan-reviewer → planner | Iterative refinement loop |
| `full-review` | scout → planner → builder → reviewer | End-to-end pipeline |

---

### Pattern 3: Parallel Research Pattern (pi-pi.ts)

The `query_experts` tool dispatches multiple experts simultaneously using `Promise.allSettled()`. All experts run as concurrent subprocesses and their results are returned together when all complete.

```
┌──────────────────────────────────────────────────────────────────┐
│               pi-pi.ts — Parallel Research Pattern              │
│                                                                  │
│  User: "Build me a Pi extension that shows CPU usage"           │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │         Primary Pi Agent (Writer + Synthesizer)         │     │
│  │                                                         │     │
│  │  Has full write tools + query_experts tool              │     │
│  │  Uses query_experts for research, then writes files     │     │
│  └───────────────────────┬─────────────────────────────────┘     │
│                          │ calls query_experts([...queries])     │
│                          │                                       │
│          Promise.allSettled() → all start simultaneously        │
│                          │                                       │
│    ┌─────────────────────┼─────────────────────────────────┐     │
│    │                     │                           │      │     │
│    ▼                     ▼                           ▼      │     │
│ ┌───────────┐      ┌───────────┐             ┌───────────┐  │     │
│ │ext-expert │      │tui-expert │             │theme-expert│ │     │
│ │           │      │           │             │           │  │     │
│ │"How do I  │      │"How do I  │             │"What color│  │     │
│ │ register  │      │ render a  │             │ tokens    │  │     │
│ │ a tool    │      │ widget?"  │             │ exist?"   │  │     │
│ │ with      │      │           │             │           │  │     │
│ │ TypeBox?" │      │pi subprocess           │pi subprocess  │     │
│ │pi subprocess      │--no-session           │--no-session   │     │
│ └─────┬─────┘      └─────┬─────┘             └─────┬─────┘  │     │
│       │                  │                         │         │     │
│       └──────────────────┼─────────────────────────┘         │     │
│                          │ all results returned together      │     │
│                          ▼                                    │     │
│  ┌─────────────────────────────────────────────────────────┐  │     │
│  │  Combined response:                                     │  │     │
│  │  ## [✓] Ext Expert (8s)                                 │  │     │
│  │  ## [✓] TUI Expert (11s)                                │  │     │
│  │  ## [✓] Theme Expert (9s)                               │  │     │
│  └─────────────────────────────────────────────────────────┘  │     │
│                          │                                    │     │
│                          ▼                                    │     │
│  Primary agent synthesizes and writes the extension files     │     │
│                                                               │     │
│  Note: pi-pi experts use --no-session (stateless queries).    │     │
│  Each query is independent — no cross-query memory.           │     │
└──────────────────────────────────────────────────────────────────┘
```

**Expert roster (from .pi/agents/pi-pi/):**

| Expert | Domain |
|---|---|
| `ext-expert` | Extension API, tools, events, commands, state management |
| `theme-expert` | Theme JSON format, color tokens, custom themes |
| `tui-expert` | Terminal UI components, overlays, widgets, footers |
| `skill-expert` | SKILL.md multi-file packages |
| `config-expert` | settings.json, providers, models |
| `prompt-expert` | Prompt template .md commands |
| `agent-expert` | Agent definition .md format, teams, orchestration |
| `keybinding-expert` | registerShortcut(), Key IDs, reserved keys |
| `cli-expert` | CLI flags, launch patterns, invocation |

---

### Pattern 4: Background Subagent Pattern (subagent-widget.ts)

Background subagents are fire-and-forget. The `/sub` command spawns a subprocess and immediately returns control to the user. When the subprocess finishes, its result is delivered as a follow-up message that triggers a new agent turn.

```
┌──────────────────────────────────────────────────────────────────┐
│           subagent-widget.ts — Background Subagent Pattern       │
│                                                                  │
│  User types: /sub list all TypeScript files and summarize them   │
│      │                                                           │
│      │ (fire and forget — user keeps working)                    │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │          spawn() → Subagent #1                          │     │
│  │          session: ~/.pi/agent/sessions/subagents/       │     │
│  │                   subagent-1-<timestamp>.jsonl          │     │
│  │                                                         │     │
│  │  ● Subagent #1  list all TypeScript files...  (0s)     │     │
│  │    Reading src/index.ts...                              │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  User types: /sub run all tests and report failures              │
│      │                                                           │
│      │ (second subagent starts immediately — both run in parallel)│
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │          spawn() → Subagent #2                          │     │
│  │  ● Subagent #2  run all tests...  (0s)                  │     │
│  │    Running jest...                                      │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Both widgets live above the editor simultaneously:             │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  ✓ Subagent #1  list TypeScript files  (23s) | Tools: 8│       │
│  │    Found 47 TypeScript files across src/             │        │
│  ├──────────────────────────────────────────────────────┤        │
│  │  ● Subagent #2  run tests  (11s) | Tools: 3          │        │
│  │    Running test suite...                             │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  When Subagent #1 finishes:                                      │
│      pi.sendMessage({                                             │
│          customType: "subagent-result",                           │
│          content: "Subagent #1 finished in 23s. Result:\n...",   │
│          display: true,                                           │
│      }, { deliverAs: "followUp", triggerTurn: true });            │
│                                                                  │
│  /subcont 1 now write tests for the utils files                  │
│      └─→ reuses subagent-1's session file (has memory)           │
└──────────────────────────────────────────────────────────────────┘
```

**Key difference from other patterns:**
- `deliverAs: "followUp"` + `triggerTurn: true` means the result is injected as a new user message that triggers the primary agent to respond.
- Session files are **persistent** — `/subcont N <prompt>` resumes the exact conversation with the subagent, passing the `-c` (continue) flag to `pi`.

---

## 10. Safety and Access Control Architecture

`damage-control.ts` implements real-time safety auditing by listening to the `tool_call` event and evaluating every tool invocation against a declarative YAML rule set before it executes.

### Rule Categories

```
┌─────────────────────────────────────────────────────────────────┐
│              damage-control-rules.yaml Structure                 │
│                                                                  │
│  bashToolPatterns:    (61 rules)                                 │
│      Regex patterns matched against bash tool `command` input.  │
│      On match: block silently, OR (ask: true) prompt user.      │
│                                                                  │
│      Examples:                                                   │
│        \brm\s+(-[^\s]*)*-[rRf]     → "rm with recursive/force" │
│        \bgit\s+reset\s+--hard\b    → "git reset --hard"         │
│        \bDROP\s+DATABASE\b         → "DROP DATABASE"            │
│        \baws\s+s3\s+rm\s+.*--recursive → "aws s3 rm --recursive"│
│        \bgcloud\s+projects\s+delete   → "gcloud projects delete" │
│                                                                  │
│  zeroAccessPaths:    (45 rules)                                  │
│      Paths that cannot be read OR written.                      │
│      Applies to: read, write, edit, grep, find, ls, bash        │
│                                                                  │
│      Examples: .env, ~/.ssh/, *.pem, *.key, ~/.aws/,            │
│                *-credentials.json, *.tfstate, dump.sql           │
│                                                                  │
│  readOnlyPaths:      (24 rules)                                  │
│      Paths that can be read but not written or modified.        │
│      Applies to: write, edit tools and bash write operations    │
│                                                                  │
│      Examples: /etc/, node_modules/, dist/, *.lock, bun.lockb,  │
│                ~/.bashrc, *.min.js, package-lock.json            │
│                                                                  │
│  noDeletePaths:      (16 rules)                                  │
│      Paths that cannot be deleted or moved.                     │
│      Applies to: bash rm/mv commands referencing these paths    │
│                                                                  │
│      Examples: .git/, LICENSE, README.md, Dockerfile,           │
│                .github/, Jenkinsfile, docker-compose.yml         │
└─────────────────────────────────────────────────────────────────┘
```

### Decision Flow

```
tool_call event fires
         │
         ▼
Is it a read/write/edit/grep/find/ls tool?
         │
         ├─ Yes → extract path(s) from input
         │           │
         │           ▼
         │        Check each path against zeroAccessPaths
         │           │
         │           ├─ Match → BLOCK immediately
         │           │          ctx.abort()
         │           │          return { block: true, reason }
         │           │
         │           └─ No match → continue
         │
         ├─ Is it write/edit?
         │           │
         │           ▼
         │        Check path against readOnlyPaths
         │           │
         │           ├─ Match → BLOCK
         │           └─ No match → continue
         │
         └─ Is it bash?
                     │
                     ▼
                  Check command against bashToolPatterns (regex)
                     │
                     ├─ Match (ask: false) → BLOCK
                     │     ctx.abort()
                     │     pi.appendEntry("damage-control-log", ...)
                     │     return { block: true, reason }
                     │
                     ├─ Match (ask: true) → PROMPT USER
                     │     confirmed = await ctx.ui.confirm(...)
                     │     if (!confirmed) → BLOCK
                     │     if (confirmed) → ALLOW
                     │
                     ├─ Check command against zeroAccessPaths (substring)
                     ├─ Check command against readOnlyPaths (heuristic)
                     ├─ Check command against noDeletePaths (rm/mv check)
                     │
                     └─ No match → return { block: false }
```

### Block Response

When a rule triggers, the blocked message instructs the LLM not to retry:

```
🛑 BLOCKED by Damage-Control: rm with recursive or force flags

DO NOT attempt to work around this restriction. DO NOT retry with
alternative commands, paths, or approaches that achieve the same
result. Report this block to the user exactly as stated and ask
how they would like to proceed.
```

---

## 11. Theme System Architecture

Themes are JSON files in `.pi/themes/`. Each file defines a `vars` block (named color values) and a `colors` block (semantic token assignments that reference vars).

### Theme JSON Structure

```json
{
  "$schema": "https://...theme-schema.json",
  "name": "synthwave",
  "vars": {
    "bg":       "#262335",
    "cyan":     "#36f9f6",
    "pink":     "#ff7edb",
    "green":    "#72f1b8"
  },
  "colors": {
    "accent":       "cyan",
    "success":      "green",
    "error":        "red",
    "warning":      "orange",
    "dim":          "comment",
    "muted":        "comment",
    "toolTitle":    "orange",
    "selectedBg":   "bgPink",
    "mdHeading":    "yellow",
    "syntaxKeyword":"red"
  }
}
```

### Theme Resolution Chain

```
Extension code:  theme.fg("accent", "some text")
                       │
                       ▼
              Token "accent" → looks up colors.accent → "cyan"
                       │
                       ▼
              "cyan" → looks up vars.cyan → "#36f9f6"
                       │
                       ▼
              Wraps text in ANSI escape: \x1b[38;2;54;249;246msome text\x1b[39m
```

### themeMap.ts — Extension Default Themes

Every extension calls `applyExtensionDefaults(import.meta.url, ctx)` in its `session_start` handler. This function:

1. Derives the extension's name from `import.meta.url` (e.g. `"tilldone"` from `extensions/tilldone.ts`).
2. Looks up the name in `THEME_MAP`.
3. Calls `ctx.ui.setTheme(themeName)`.
4. Sets the terminal window title to `π - <extension-name>`.

When multiple extensions are stacked, only the first extension in the `-e` argument list sets the theme (to avoid the last-loaded extension overwriting the primary extension's theme).

```
THEME_MAP = {
    "agent-chain":        "midnight-ocean",
    "agent-team":         "dracula",
    "cross-agent":        "ocean-breeze",
    "damage-control":     "gruvbox",
    "minimal":            "synthwave",
    "pi-pi":              "rose-pine",
    "pure-focus":         "everforest",
    "purpose-gate":       "tokyo-night",
    "session-replay":     "catppuccin-mocha",
    "subagent-widget":    "cyberpunk",
    "system-select":      "catppuccin-mocha",
    "theme-cycler":       "synthwave",
    "tilldone":           "everforest",
    "tool-counter":       "synthwave",
    "tool-counter-widget":"synthwave",
}
```

---

## 12. Agent Definition Format

Agent definitions are Markdown files with YAML frontmatter. They are used by `agent-team.ts`, `agent-chain.ts`, `system-select.ts`, and `cross-agent.ts`.

### Format

```markdown
---
name: builder
description: Implementation and code generation
tools: read,write,edit,bash,grep,find,ls
---
You are a builder agent. Implement the requested changes thoroughly.
Write clean, minimal code. Follow existing patterns in the codebase.
Test your work when possible.
```

### Frontmatter Fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Agent identifier (lowercase, hyphenated) |
| `description` | No | One-line description shown in team/chain selection dialogs |
| `tools` | No | Comma-separated list of tools this agent can use. Defaults to `read,grep,find,ls`. |

The body below the `---` closing delimiter becomes the agent's system prompt, appended to Pi's default system prompt via `--append-system-prompt`.

### Discovery Directories

Extensions scan these directories in order, deduplicating by name:

1. `<cwd>/agents/`
2. `<cwd>/.claude/agents/`
3. `<cwd>/.pi/agents/`
4. `~/.claude/agents/` (global)
5. `~/.pi/agent/agents/` (global)

### Team Definitions (teams.yaml)

```yaml
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

### Chain Definitions (agent-chain.yaml)

```yaml
plan-build-review:
  description: "Plan, implement, and review — the standard development cycle"
  steps:
    - agent: planner
      prompt: "Plan the implementation for: $INPUT"
    - agent: builder
      prompt: "Implement the following plan:\n\n$INPUT"
    - agent: reviewer
      prompt: "Review this implementation for bugs, style, and correctness:\n\n$INPUT"
```

---

## 13. Session and State Management

### Session Branching Model

Pi maintains a tree of session branches. Each branch is a linear list of messages. The user can fork a branch (creating a new branch from the current point), switch between branches, and view the branch tree. Extensions can reconstruct their state from the current branch's message history.

```
Branch Tree:
                    ┌─ [main] ─────────────────────────────────┐
                    │  msg1 (user)                              │
                    │  msg2 (assistant)                        │
                    │  msg3 (toolResult: tilldone)             │
                    │  msg4 (user: fork point)                 │
                    └────────────────┬────────────────────────┘
                                     │ fork
                           ┌─────────┴──────────┐
                           │                    │
              ┌─ [branch-a] ─┐      ┌─ [branch-b] ─┐
              │  msg5        │      │  msg5b        │
              │  msg6        │      │  msg6b        │
              └──────────────┘      └───────────────┘
```

### State Reconstruction Pattern

Extensions that need to survive branch switches (like `tilldone.ts`) reconstruct their state by replaying the current branch's tool results:

```typescript
const reconstructState = (ctx: ExtensionContext) => {
    tasks = [];
    nextId = 1;

    // Walk all messages in the current branch
    for (const entry of ctx.sessionManager.getBranch()) {
        if (entry.type !== "message") continue;
        const msg = entry.message;

        // Find all tool results from our tool
        if (msg.role !== "toolResult" || msg.toolName !== "tilldone") continue;

        // The details field holds our serialized state snapshot
        const details = msg.details as TillDoneDetails | undefined;
        if (details) {
            tasks = details.tasks;
            nextId = details.nextId;
        }
    }

    refreshUI(ctx);
};

// Register on all branch-change events
pi.on("session_start",  async (_event, ctx) => reconstructState(ctx));
pi.on("session_switch", async (_event, ctx) => reconstructState(ctx));
pi.on("session_fork",   async (_event, ctx) => reconstructState(ctx));
pi.on("session_tree",   async (_event, ctx) => reconstructState(ctx));
```

The `details` field in a tool result is an arbitrary object that extensions attach to their tool's return value. Pi serializes it to the session file, making it available for reconstruction.

### Subagent Session Files

Multi-agent extensions store subprocess session files in `.pi/agent-sessions/`:

| Extension | Session File Pattern | Persistence |
|---|---|---|
| `agent-team.ts` | `.pi/agent-sessions/<agent-name>.json` | Wiped on each `session_start` |
| `agent-chain.ts` | `.pi/agent-sessions/chain-<agent-name>.json` | Wiped on each `session_start` |
| `pi-pi.ts` | No session files | `--no-session` flag — each query is stateless |
| `subagent-widget.ts` | `~/.pi/agent/sessions/subagents/subagent-<id>-<ts>.jsonl` | Persistent across `/subcont` calls |

---

## 14. Communication Flow Diagrams

### Flow 1: Simple Direct Request

```
User types a prompt
        │
        ▼
pi.on("input") fires
        │
        ├─ purpose-gate: is purpose set? No → { action: "handled" }, show warning
        ├─ tilldone: no effect on input (only tool_call is gated)
        └─ default: { action: "continue" }
        │
        ▼
pi.on("before_agent_start") fires
        │
        ├─ agent-team: inject dispatcher system prompt
        ├─ agent-chain: inject pipeline system prompt
        ├─ purpose-gate: append <purpose> to system prompt
        └─ system-select: prepend selected agent's body
        │
        ▼
pi.on("agent_start") fires
        │
        ▼
LLM processes prompt + system prompt + history
        │
        ▼
pi.on("message_start") fires
        │
        ▼
LLM generates response text
        │
        ▼
pi.on("message_update") fires (many times, one per text delta)
        │
        ▼
pi.on("message_end") fires → usage/cost captured
        │
        ▼
pi.on("agent_end") fires
        │
        ├─ tilldone: are there incomplete tasks? → inject nudge message
        └─ no other effects
```

### Flow 2: Tool Call with Damage Control

```
LLM decides to call bash("rm -rf /tmp/build")
        │
        ▼
pi.on("tool_call") fires with event.toolName = "bash"
        │
        ▼
damage-control handler:
    isToolCallEventType("bash", event) → true
    regex test: \brm\s+(-[^\s]*)*-[rRf] matches "rm -rf"
        │
        ├─ rule has ask: false
        │        │
        │        ▼
        │   ctx.abort()
        │   ctx.ui.notify("🛑 Blocked rm with recursive or force flags")
        │   pi.appendEntry("damage-control-log", { tool, input, rule, action: "blocked" })
        │   return { block: true, reason: "🛑 BLOCKED by Damage-Control: ..." }
        │
        └─ rule has ask: true (e.g. git checkout -- .)
                 │
                 ▼
            confirmed = await ctx.ui.confirm(
                "🛡️ Damage-Control Confirmation",
                "Dangerous command detected: ..."
            )
                 │
                 ├─ user says No → ctx.abort(), return { block: true }
                 └─ user says Yes → return { block: false }
```

### Flow 3: Dispatcher Pattern Request

```
User: "Add JWT authentication to the auth module"
        │
        ▼
Dispatcher agent (only has dispatch_agent tool)
        │
        ▼
LLM thinks: "I need to explore first, then plan, then implement"
        │
        ▼
dispatch_agent({ agent: "scout", task: "Find the auth module and report structure" })
        │
        ▼
dispatchAgent() spawns: pi --mode json -p --no-extensions --tools read,grep,find,ls
                            --session .pi/agent-sessions/scout.json
                            "Find the auth module..."
        │
        ├─ subprocess streams JSON events → widget updates live
        └─ subprocess exits → output returned to dispatcher
        │
        ▼
dispatch_agent({ agent: "planner", task: "Plan JWT auth based on:\n\n<scout output>" })
        │
        ▼
dispatch_agent({ agent: "builder", task: "Implement:\n\n<planner output>" })
        │
        ▼
Dispatcher synthesizes results and responds to user
```

### Flow 4: Session Replay Overlay

```
User types: /replay
        │
        ▼
pi.registerCommand("replay").handler fires
        │
        ▼
ctx.sessionManager.getBranch()  ← read all messages in current branch
        │
        ▼
Build HistoryItem[] from branch messages
        │
        ▼
await ctx.ui.custom(factory, { overlay: true, overlayOptions: { width: "80%", anchor: "center" } })
        │
        ▼ (blocks until onDone() called)
┌─────────────────────────────────────────────────────────┐
│  SESSION REPLAY  |  12 entries                          │
│ ─────────────────────────────────────────────────────── │
│ 👤 User Prompt  [10:23:01]                              │
│   implement authentication...                           │
│ 🤖 Assistant  [10:23:08] (+7s)                          │
│   I'll start by reading the existing auth files...      │
│ 🛠️ Tool: read  [10:23:09] (+1s)         ← selected      │
│   Content of src/auth/index.ts:                        │
│   import jwt from 'jsonwebtoken'...                    │
│                                                         │
│ ↑/↓ Navigate • Enter Expand • Esc Close                │
│ ─────────────────────────────────────────────────────── │
└─────────────────────────────────────────────────────────┘
        │ user presses Esc
        ▼
ctx.ui.custom() returns → /replay handler returns
        │
        ▼
Normal interaction resumes
```

### Flow 5: Theme Cycling

```
User presses Ctrl+X
        │
        ▼
pi.registerShortcut("ctrl+x").handler fires
        │
        ▼
cycleTheme(ctx, +1)
        │
        ▼
ctx.ui.getAllThemes()  ← returns array of { name, path } objects
        │
        ▼
find current index → increment → wrap around
        │
        ▼
ctx.ui.setTheme(themes[newIndex].name)
        │
        ▼
ctx.ui.setStatus("theme", "🎨 tokyo-night")
        │
        ▼
Show swatch widget for 3 seconds:
    ███ ███ ███ ███ ███   (colored blocks in success/accent/warning/dim/muted)
        │
        ▼
setTimeout(3000) → ctx.ui.setWidget("theme-swatch", undefined)
```

---

## 15. Data Flow: How a User Prompt Becomes an Action

This traces the full path from keypress to tool execution for a request handled by `agent-chain.ts`:

```
┌────────────────────────────────────────────────────────────────┐
│  1. INPUT                                                      │
│                                                                │
│     User types: "Add error handling to the API routes"         │
│     Presses Enter                                              │
│     Pi captures raw keystrokes from stdin                      │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  2. INPUT EVENT                                                │
│                                                                │
│     pi.on("input") fires                                       │
│     agent-chain: no gate, returns { action: "continue" }       │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  3. BEFORE AGENT START                                         │
│                                                                │
│     pi.on("before_agent_start") fires                          │
│     agent-chain injects system prompt:                         │
│       "You are an agent with pipeline 'plan-build-review'...   │
│        Flow: Planner → Builder → Reviewer...                   │
│        Use run_chain for significant work..."                  │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  4. LLM CALL                                                   │
│                                                                │
│     pi-agent-core sends to LLM:                                │
│       - system prompt (Pi default + chain injection)           │
│       - conversation history from current branch               │
│       - available tools: all built-in + run_chain              │
│       - user message: "Add error handling to the API routes"   │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  5. LLM DECISION                                               │
│                                                                │
│     LLM decides this is "significant work" → use run_chain     │
│     LLM emits tool call: run_chain({ task: "Add error         │
│       handling to all route handlers in src/routes/" })        │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  6. TOOL CALL EVENT                                            │
│                                                                │
│     pi.on("tool_call") fires                                   │
│     damage-control: checks run_chain → no matching rules       │
│     returns { block: false }                                   │
│     tilldone: is there an inprogress task? (if loaded)         │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  7. TOOL EXECUTION                                             │
│                                                                │
│     run_chain.execute() is called                              │
│     Step 1: spawn pi for planner                               │
│       prompt: "Plan the implementation for: Add error          │
│                handling to all route handlers in src/routes/"  │
│       → subprocess runs, reads codebase, outputs plan         │
│                                                                │
│     Step 2: spawn pi for builder                               │
│       prompt: "Implement the following plan:\n\n<plan output>" │
│       → subprocess runs, reads + writes files                  │
│                                                                │
│     Step 3: spawn pi for reviewer                              │
│       prompt: "Review this implementation:\n\n<build output>"  │
│       → subprocess runs, reviews, outputs report              │
│                                                                │
│     All three widget cards update in real time during          │
│     execution (Planner → Builder → Reviewer status arrows)     │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  8. TOOL RESULT                                                │
│                                                                │
│     run_chain returns to pi-agent-core:                        │
│       content: "[chain:plan-build-review] done in 87s\n\n..."  │
│     pi-agent-core appends as toolResult message to branch      │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  9. FINAL LLM CALL                                             │
│                                                                │
│     LLM sees the tool result and generates a summary           │
│     for the user: "I've completed the error handling           │
│     implementation. Here's what was done..."                   │
└─────────────────────────────────────┬──────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────┐
│  10. AGENT END                                                 │
│                                                                │
│     pi.on("agent_end") fires                                   │
│     tilldone nudge check (if loaded)                           │
│     footer/widget refresh                                      │
└────────────────────────────────────────────────────────────────┘
```

---

## 16. Cross-Agent Integration

`cross-agent.ts` discovers and bridges assets from other AI coding agent ecosystems into Pi. It scans for commands, skills, and agents from Claude Code, Gemini CLI, and Codex configurations.

### Discovery Scan Pattern

```
cross-agent.ts on session_start:

  Providers: ["claude", "gemini", "codex"]
  Locations: [project local, global ~/.provider]

  For each combination:
    .claude/commands/*.md   → register as Pi /command
    .claude/skills/         → discover skill names
    .claude/agents/*.md     → discover agent names
    .pi/agents/*.md         → discover Pi-native agent names
    ~/.claude/commands/*.md → register as Pi /command (global)
    ... (same for gemini, codex)
```

### Command Registration

When a `.md` file is found in a `commands/` directory, its body is used as a prompt template. The `/command-name` slash command is registered in Pi. When invoked, `sendUserMessage()` fires the template with argument substitution:

```typescript
pi.registerCommand(cmd.name, {
    description: `[${g.source}] ${cmd.description}`,
    handler: async (args) => {
        // Substitute $ARGUMENTS/$@ and $1..$N positional args
        pi.sendUserMessage(expandArgs(cmd.content, args || ""));
    },
});
```

This means a Claude Code slash command like `/prime` (which primes the agent with context) becomes available in Pi without any modification to the `.md` file.

### Argument Expansion

```typescript
function expandArgs(template: string, args: string): string {
    const parts = args.split(/\s+/).filter(Boolean);
    let result = template;
    result = result.replace(/\$ARGUMENTS|\$@/g, args);  // whole arg string
    for (let i = 0; i < parts.length; i++) {
        result = result.replaceAll(`$${i + 1}`, parts[i]);  // $1, $2, ...
    }
    return result;
}
```

### Discovery Summary Notification

On startup, a styled notification shows all discovered assets grouped by source directory, using raw ANSI codes (synthwave palette):

```
  .claude                         (1 command)
  ┌────────────────────────────────────────┐
  │  /prime                                │
  └────────────────────────────────────────┘

  .pi/agents                      (8 agents)
  ┌────────────────────────────────────────┐
  │  @planner, @builder, @scout, @reviewer │
  │  @documenter, @red-team, ...           │
  └────────────────────────────────────────┘
```

---

## 17. Key Code Patterns and Idioms

### Pattern 1: Extension Entry Point

Every extension follows the same module structure. The default export is the entry point.

```typescript
// extensions/my-extension.ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";

export default function (pi: ExtensionAPI) {
    // Register tools, commands, shortcuts SYNCHRONOUSLY at the top level
    pi.registerTool({ name: "my-tool", ... });
    pi.registerCommand("my-cmd", { ... });

    // Subscribe to events
    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);  // theme + title
        // ... initialize UI
    });
}
```

The critical rule: **tools must be registered at the top level**, not inside event handlers. Pi reads the tool registry before the session starts to build the LLM's tool schema.

### Pattern 2: setFooter Factory

The footer factory is called once; it returns a renderer that Pi calls every time it needs to redraw the footer. The renderer captures its dependencies via closure.

```typescript
// From minimal.ts
ctx.ui.setFooter((_tui, theme, _footerData) => ({
    dispose: () => {},
    invalidate() {},
    render(width: number): string[] {
        // ctx is captured from the outer scope
        const model = ctx.model?.id || "no-model";
        const usage = ctx.getContextUsage();
        const pct = usage?.percent ?? 0;
        const filled = Math.round(pct / 10);
        const bar = "#".repeat(filled) + "-".repeat(10 - filled);

        const left = theme.fg("dim", ` ${model}`);
        const right = theme.fg("dim", `[${bar}] ${Math.round(pct)}% `);
        const pad = " ".repeat(Math.max(1, width - visibleWidth(left) - visibleWidth(right)));

        return [truncateToWidth(left + pad + right, width)];
    },
}));
```

The `footerData.onBranchChange(() => tui.requestRender())` pattern is used when the footer displays git branch information that can change mid-session.

### Pattern 3: setWidget with Dynamic Content

Widgets are placed in the "belowEditor" zone. Each widget is a named slot — calling `setWidget` with the same name replaces the previous widget; calling it with `undefined` removes the widget.

```typescript
// From tilldone.ts — widget that shows current in-progress task
ctx.ui.setWidget("tilldone-current", (_tui, theme) => {
    const container = new Container();
    const borderFn = (s: string) => theme.fg("dim", s);

    container.addChild(new Text("", 0, 0));       // top margin
    container.addChild(new DynamicBorder(borderFn));
    const content = new Text("", 1, 0);           // content line
    container.addChild(content);
    container.addChild(new DynamicBorder(borderFn));

    return {
        render(width: number): string[] {
            // Re-read live state on every render call
            const cur = tasks.find((t) => t.status === "inprogress");
            if (!cur) return [];

            const line =
                theme.fg("accent", "● ") +
                theme.fg("dim", "WORKING ON  ") +
                theme.fg("accent", `#${cur.id}`) +
                theme.fg("success", cur.text);

            content.setText(truncateToWidth(line, width - 4));
            return container.render(width);
        },
        invalidate() { container.invalidate(); },
    };
}, { placement: "belowEditor" });
```

### Pattern 4: Blocking Tool Calls

The `tool_call` event handler can return `{ block: true, reason }` to prevent any tool from executing. The `reason` string is sent back to the LLM as if it were a tool error.

```typescript
// From tilldone.ts — block all tools unless a task is in-progress
pi.on("tool_call", async (event, _ctx) => {
    if (event.toolName === "tilldone") return { block: false };  // always allow our own tool

    const pending = tasks.filter((t) => t.status !== "done");
    const active  = tasks.filter((t) => t.status === "inprogress");

    if (tasks.length === 0) {
        return {
            block: true,
            reason: "No TillDone tasks defined. You MUST use `tilldone new-list` " +
                    "or `tilldone add` before using any other tools.",
        };
    }
    if (active.length === 0) {
        return {
            block: true,
            reason: "No task is in progress. Use `tilldone toggle` to mark a task inprogress.",
        };
    }

    return { block: false };
});
```

### Pattern 5: System Prompt Injection

The `before_agent_start` event fires before the LLM is called on each turn. Returning `{ systemPrompt }` replaces the system prompt for that turn. This is how `agent-team.ts` builds a dynamic agent catalog:

```typescript
pi.on("before_agent_start", async (_event, _ctx) => {
    // Build catalog from currently active team members
    const agentCatalog = Array.from(agentStates.values())
        .map(s => `### ${displayName(s.def.name)}\n**Dispatch as:** \`${s.def.name}\`\n${s.def.description}`)
        .join("\n\n");

    return {
        systemPrompt: `You are a dispatcher agent...

## Active Team: ${activeTeamName}
## Agents
${agentCatalog}`,
    };
});
```

### Pattern 6: Custom Tool with renderCall/renderResult

Tools can supply custom terminal UI for their call display and result display:

```typescript
// From agent-team.ts
pi.registerTool({
    name: "dispatch_agent",
    description: "Dispatch a task to a specialist agent.",
    parameters: Type.Object({
        agent: Type.String({ description: "Agent name" }),
        task:  Type.String({ description: "Task description" }),
    }),

    async execute(_toolCallId, params, _signal, onUpdate, ctx) {
        // Tool implementation
    },

    // Shown while the tool call is pending (before execution)
    renderCall(args, theme) {
        return new Text(
            theme.fg("toolTitle", theme.bold("dispatch_agent ")) +
            theme.fg("accent", args.agent) +
            theme.fg("dim", " — ") +
            theme.fg("muted", args.task.slice(0, 60)),
            0, 0,
        );
    },

    // Shown after execution; receives options.expanded for detail toggle
    renderResult(result, options, theme) {
        const details = result.details as any;
        const icon = details.status === "done" ? "✓" : "✗";
        const color = details.status === "done" ? "success" : "error";

        if (options.expanded && details.fullOutput) {
            return new Text(
                theme.fg(color, `${icon} ${details.agent}`) + "\n" +
                theme.fg("muted", details.fullOutput.slice(0, 4000)),
                0, 0,
            );
        }

        return new Text(
            theme.fg(color, `${icon} ${details.agent}`) +
            theme.fg("dim", ` ${Math.round(details.elapsed / 1000)}s`),
            0, 0,
        );
    },
});
```

### Pattern 7: StringEnum for Google Gemini Compatibility

Google's Gemini models do not support plain string enums in JSON Schema. The `StringEnum` helper from `@mariozechner/pi-ai` works around this by generating a compatible schema:

```typescript
// From tilldone.ts
import { StringEnum } from "@mariozechner/pi-ai";
import { Type } from "@sinclair/typebox";

const TillDoneParams = Type.Object({
    action: StringEnum([
        "new-list", "add", "toggle", "remove", "update", "list", "clear"
    ] as const),
    text: Type.Optional(Type.String({ description: "Task text" })),
    id:   Type.Optional(Type.Number({ description: "Task ID" })),
});
```

Using `Type.Union([Type.Literal("add"), Type.Literal("remove"), ...])` instead would work for some providers but break with Gemini.

### Pattern 8: State Stored in Tool Result Details

Rather than using a global variable that would be lost on branch switch, extensions can persist state in tool result `details`. Pi serializes these to the session file, so they survive process restarts and branch switches:

```typescript
// Execute returns details attached to the tool result message
async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
    // ... mutate tasks array ...

    return {
        content: [{ type: "text", text: `Added task #${task.id}` }],
        details: {      // ← serialized to session, read back in reconstructState()
            action: "add",
            tasks: [...tasks],       // current task list snapshot
            nextId,
            listTitle,
            listDescription,
        },
    };
},
```

### Pattern 9: applyExtensionDefaults for Theme and Title

Every extension calls this single function in `session_start` to apply its mapped theme and set the terminal window title. It handles the stacking case correctly (only the first extension in the argument list sets the theme):

```typescript
// In every extension's session_start handler:
pi.on("session_start", async (_event, ctx) => {
    applyExtensionDefaults(import.meta.url, ctx);
    // import.meta.url is "file:///path/to/extensions/my-extension.ts"
    // applyExtensionDefaults derives "my-extension" from it, looks up THEME_MAP
    // ... rest of initialization
});
```

### Pattern 10: isToolCallEventType for Type-Safe Event Narrowing

The `isToolCallEventType` type guard narrows the generic tool_call event to a tool-specific typed event, providing autocomplete and compile-time safety for the `event.input` shape:

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
    // event.toolName is string, event.input is unknown

    if (isToolCallEventType("bash", event)) {
        // Now: event.input.command is string
        const command = event.input.command;
        for (const rule of rules.bashToolPatterns) {
            if (new RegExp(rule.pattern).test(command)) {
                return { block: true, reason: rule.reason };
            }
        }
    }

    if (isToolCallEventType("write", event)) {
        // Now: event.input.path is string, event.input.content is string
        const resolvedPath = path.resolve(ctx.cwd, event.input.path);
        // ... check readOnlyPaths
    }

    return { block: false };
});
```
