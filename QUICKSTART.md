# Quick Start Guide — Pi vs Claude Code Extension Playground

This guide is written for developers who are comfortable with C, C++, or Java but are
new to TypeScript, terminal-based AI tools, and modern JavaScript tooling. Every step
is explained from scratch. No prior knowledge of AI APIs or Node.js is assumed.

---

## Table of Contents

1. [What Is This Project?](#1-what-is-this-project)
2. [Prerequisites — Installing Everything From Scratch](#2-prerequisites--installing-everything-from-scratch)
3. [Setting Up API Keys](#3-setting-up-api-keys)
4. [Installing the Project](#4-installing-the-project)
5. [Ten Quick-Win Use Cases](#5-ten-quick-win-use-cases)
6. [Understanding What You See On Screen](#6-understanding-what-you-see-on-screen)
7. [Commands Cheat Sheet](#7-commands-cheat-sheet)
8. [Keyboard Shortcuts](#8-keyboard-shortcuts)
9. [Common Troubleshooting](#9-common-troubleshooting)
10. [What's Next?](#10-whats-next)

---

## 1. What Is This Project?

### Pi Coding Agent

**Pi Coding Agent** is a terminal-based AI coding assistant. Think of it like having a
senior developer sitting at your terminal: you type a task in plain English, and it reads
your files, writes code, runs shell commands, and reports what it did — all without leaving
the terminal.

If you have heard of **Claude Code** (Anthropic's official terminal AI tool), Pi is the
same category of product. The key difference is that Pi is **open-source** and has a rich
**extension system** that lets you customize almost everything about how it looks and
behaves.

### This Repository

This repository is a **collection of 16 custom extensions** that demonstrate what the Pi
extension system can do. Each extension is a single TypeScript file that plugs into Pi
and adds new behavior:

| Category | What the extensions show |
|---|---|
| **UI customization** | Strip the interface down to nothing; add rich token/cost meters; add live progress widgets |
| **Workflow enforcement** | Force a declared purpose before work begins; require task planning before any tools run |
| **Multi-agent orchestration** | Spawn background worker agents; run teams of specialists in parallel; chain agents in pipelines |
| **Safety controls** | Block dangerous shell commands; protect sensitive files from being read or deleted |
| **Meta-programming** | Use Pi to build new Pi extensions using parallel expert subagents |

You do not need to understand TypeScript deeply to run and experiment with these extensions.
The `just` task runner wraps every extension into a single command like `just ext-minimal`.

---

## 2. Prerequisites — Installing Everything From Scratch

You need three tools before anything else will work. Think of them as the compiler,
linker, and build system for this project.

### 2.1 Bun — Runtime and Package Manager

**What it is:** Bun is a JavaScript/TypeScript runtime (like a Java JVM, but for JS/TS)
combined with a package manager (like Maven or apt-get, but for npm packages). It is
significantly faster than the traditional Node.js + npm combination.

**Why you need it:** Pi extensions are TypeScript files. Bun runs them. `bun install`
downloads the project's declared dependencies — the same concept as `make` pulling
libraries or `mvn install` fetching jars.

**Install instructions:**

macOS and Linux (run in your terminal):
```bash
curl -fsSL https://bun.sh/install | bash
```

Windows (run in PowerShell as administrator):
```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

After installation, close and reopen your terminal, then verify:
```bash
bun --version
# Expected output: 1.x.x (some version number)
```

If you see a version number, Bun is installed correctly.

**Full documentation:** https://bun.sh

---

### 2.2 just — Task Runner

**What it is:** `just` is a command runner. It reads a file called `justfile` (similar to
a `Makefile`) and lets you run named recipes with a short command. Instead of memorizing
`pi -e extensions/minimal.ts -e extensions/theme-cycler.ts`, you type `just ext-minimal`.

**Why you need it:** The `justfile` in this project also automatically loads your `.env`
file (your API keys) before every command. Without `just`, you would need to load the keys
manually each time.

**Install instructions:**

macOS (using Homebrew):
```bash
brew install just
```

Linux (using Cargo, Rust's package manager):
```bash
cargo install just
```

Linux (pre-built binary, no Rust needed):
```bash
# Download the latest binary for your architecture from:
# https://github.com/casey/just/releases
# Example for x86_64 Linux:
curl -L https://github.com/casey/just/releases/latest/download/just-x86_64-unknown-linux-musl.tar.gz | tar xz
sudo mv just /usr/local/bin/
```

Windows (using Scoop):
```powershell
scoop install just
```

Windows (using Winget):
```powershell
winget install Casey.Just
```

After installation, verify:
```bash
just --version
# Expected output: just x.x.x
```

**Full documentation:** https://just.systems

---

### 2.3 Pi Coding Agent — The AI Assistant CLI

**What it is:** Pi is the AI assistant itself. It is distributed as an npm package and
installed globally so that the `pi` command is available everywhere on your system.
Think of it like installing a compiler: once installed, you can invoke it from any directory.

**Why you need it:** Every extension in this repository is loaded by Pi. Without Pi,
there is nothing to run the extensions.

**Install instructions:**

```bash
npm install -g @mariozechner/pi-coding-agent
```

Note: This uses `npm` (which comes bundled with Node.js), not `bun`. Pi itself is
distributed via the npm registry. If you do not have Node.js installed:

- macOS: `brew install node`
- Linux: `sudo apt install nodejs npm` (Ubuntu/Debian) or `sudo dnf install nodejs npm` (Fedora)
- Windows: Download from https://nodejs.org

After installation, verify:
```bash
pi --version
# Expected output: pi x.x.x
```

**Official Pi documentation:** https://github.com/mariozechner/pi-coding-agent

---

### Summary Checklist

Run all three verification commands before continuing. All three must succeed:

```bash
bun --version     # Must print a version number
just --version    # Must print a version number
pi --version      # Must print a version number
```

---

## 3. Setting Up API Keys

### What Is an API Key?

An API key is a secret credential — similar to a password — that identifies you to an AI
provider's service and authorizes your requests. When Pi sends a prompt to an AI model
(such as Claude or GPT-4), it attaches your API key so the provider knows which account
to bill and which permissions to apply.

**Keep your API keys private.** Never commit them to version control. The `.env` file
(where you store keys) is listed in `.gitignore` for exactly this reason.

### Step 1: Copy the Sample File

The repository includes `.env.sample`, a template showing exactly which keys are needed.
Copy it to create your own private `.env` file:

```bash
cp .env.sample .env
```

Now open `.env` in any text editor. It looks like this:

```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
OPENROUTER_API_KEY=sk-or-...
FIRECRAWL_API_KEY=fc-...
```

Replace each `...` placeholder with your real key. You only need **one** key to get
started. Anthropic (Claude) or OpenRouter are good choices for beginners.

### Step 2: Get Your Keys

You only need one provider to get started. OpenRouter is the easiest because it provides
access to many models through a single key.

| Provider | What models you get | Where to get a key |
|---|---|---|
| **OpenAI** | GPT-4o, o1, o3 | https://platform.openai.com/api-keys |
| **Anthropic** | Claude Sonnet, Claude Opus | https://console.anthropic.com/settings/keys |
| **Google** | Gemini 2.0, Gemini 1.5 | https://aistudio.google.com/app/apikey |
| **OpenRouter** | All of the above + more, via one key | https://openrouter.ai/keys |
| **Firecrawl** | Web scraping (used by pi-pi extension only) | https://firecrawl.dev |

For each provider: create an account, navigate to the API keys section, generate a new
key, and paste it into your `.env` file.

### Step 3: Load the Keys Into Your Shell

API keys must be present as **environment variables** in your shell before Pi starts.
An environment variable works like a global constant visible to every process launched
from that shell — similar to a `#define` that every program in your build can see.

There are three ways to load the keys:

**Option A — Source manually (simplest, works every time):**
```bash
source .env && pi
```
The `source` command reads the `.env` file and exports each variable into your current
shell session. You must repeat this each time you open a new terminal.

**Option B — Shell alias (convenient for daily use):**

Add this line to your `~/.bashrc` or `~/.zshrc`:
```bash
alias pi='source /path/to/pi-vs-claude-code-main/.env && pi'
```

Replace `/path/to/pi-vs-claude-code-main` with the actual path to where you cloned the
repository. After adding the alias, reload your shell config:
```bash
source ~/.bashrc   # or source ~/.zshrc
```

Now typing `pi` anywhere will automatically load your keys first.

**Option C — Use `just` (recommended; automatic for all recipes):**
```bash
just pi           # Runs plain Pi — .env is loaded automatically
just ext-minimal  # Runs the minimal extension — .env is loaded automatically
```

The first line of the `justfile` is `set dotenv-load := true`, which tells `just` to
automatically read the `.env` file before running any recipe. This is the most
convenient approach and the one used throughout this guide.

---

## 4. Installing the Project

### Clone the Repository

If you have not already downloaded the project files:

```bash
git clone https://github.com/your-org/pi-vs-claude-code-main.git
cd pi-vs-claude-code-main
```

Replace the URL with the actual repository URL if different.

### Install Dependencies

```bash
bun install
```

This command reads `package.json` (the project's dependency manifest — analogous to a
`pom.xml` in Maven or a `CMakeLists.txt`) and downloads all declared packages into the
`node_modules/` directory. In this project there is only one dependency (`yaml`), so this
runs very quickly.

This is equivalent to running `make` to compile dependencies, or `mvn install` to
download jars into your local Maven repository.

### Verify the Setup

Run `just` with no arguments to see a list of all available recipes:

```bash
just
```

You should see a table listing all the extension commands (`ext-minimal`,
`ext-tool-counter`, `ext-agent-team`, etc.). If you see this list, the setup is complete.

---

## 5. Ten Quick-Win Use Cases

Each use case below takes under five minutes. They are ordered from simplest to most
advanced. Run them in order on your first session.

---

### Use Case 1: Your First Pi Session

**Command:**
```bash
just pi
```

**What it does:** Launches Pi with no extensions — the bare AI assistant in its default
configuration.

**What to do:** After Pi starts, type any coding question or request in the input area
at the bottom of the screen and press Enter. For example:

```
Write a function in C that reverses a string in place.
```

Pi will respond with code and an explanation. Watch the screen as it types the response.

**What you are seeing:** Pi's default terminal user interface (TUI). There is a header at
the top showing the model name and session ID, a scrollable response area in the middle,
an input editor at the bottom, and a status line at the very bottom showing context usage.

**Key concept — context window:** The AI has a limited "working memory" called the context
window, measured in tokens (roughly 0.75 words per token). Every message you send and every
response you receive consumes tokens. When the window fills up, Pi automatically compresses
older history to make room. The status line shows how full the context window currently is.

---

### Use Case 2: Distraction-Free Coding

**Command:**
```bash
just ext-pure-focus
```

**What it does:** Loads the `pure-focus` extension, which removes the footer bar and
the status line entirely — leaving only the response area and the input editor.

**What to try:** Ask Pi to do something, then compare the screen to Use Case 1.

**Why this matters:** This extension demonstrates that Pi's UI elements are not hard-coded.
An extension can add, remove, or replace any part of the interface. If you find the status
line distracting during long coding sessions, this extension strips it away.

**The extension in one sentence:** It intercepts the `session_start` event and calls
`ctx.ui.setFooter(null)` to suppress the default footer.

---

### Use Case 3: See What Your AI Is Thinking

**Command:**
```bash
just ext-minimal
```

**What it does:** Loads the `minimal` extension, which replaces the default footer with
a compact one-line display showing the active model name and a 10-block context usage bar:

```
claude-sonnet-4-5        [###-------] 30%
```

**What to watch:** As you send messages and receive responses, watch the bar fill up from
left to right. Each `#` represents roughly 10% of the context window.

**Why this matters:** Context windows have real costs. A nearly full context window means
the AI is carrying a lot of conversation history, which makes each request more expensive.
The context bar gives you a concrete visual signal of when to start a fresh session.

**Analogy for C/Java developers:** Think of the context window as a fixed-size ring buffer
on the heap. The bar shows you how full the buffer is. When it overflows, the oldest
entries are dropped (or summarized).

---

### Use Case 4: Track Every Tool the AI Uses

**Command:**
```bash
just ext-tool-counter
```

**What it does:** Replaces the footer with a rich two-line display:

```
claude-sonnet-4-5  [###-------] 30%          12.4k in  2.1k out  $0.0032
my-project (main)                  read 4 | bash 2 | grep 1
```

- **Line 1:** Model name, context bar, total tokens sent to the AI (`in`), total tokens
  received (`out`), and total cost in US dollars for this session.
- **Line 2:** Current working directory and git branch on the left; per-tool call counts
  on the right.

**What to watch:** Ask Pi to read a file or run a bash command, then watch the tool tally
update in real time. Each tool call increments its counter.

**Why this matters:** AI API costs are metered per token. This extension makes the cost
of each interaction visible. A request that causes 10 tool calls is more expensive than
one that causes 2. Use this extension when you want to understand and optimize what the
AI is actually doing.

---

### Use Case 5: Stay Focused With Purpose Gate

**Command:**
```bash
just ext-purpose-gate
```

**What it does:** When Pi starts, it immediately pops up a dialog asking:

```
What is the purpose of this agent?
e.g. Refactor the auth module to use JWT
```

You must type an answer and press Enter before you can use Pi at all. Once you answer,
a persistent banner appears at the top of the editor showing your stated purpose for the
rest of the session.

**What to try:** Start Pi, declare a purpose like "Add input validation to the login
form", then try asking Pi something completely unrelated. Watch it gently remind you of
your stated goal.

**Why this matters:** AI assistants are very helpful but can easily lead you down
rabbit holes. This extension enforces a lightweight discipline: you must commit to a goal
before any work begins. The purpose is also injected into the AI's system prompt, so
the AI itself stays focused on your goal.

**Key insight:** Extensions can intercept the `input` event and return `{ action: "handled" }`
to block a message from reaching the AI. This extension uses that mechanism to block all
input until a purpose is set.

---

### Use Case 6: Task Discipline With TillDone

**Command:**
```bash
just ext-tilldone
```

**What it does:** Enforces a structured workflow. The AI **cannot use any tools** (cannot
read files, cannot run commands) until it has first used the `tilldone` tool to define
a task list. After defining tasks, the AI must mark each task "in progress" before working
on it, and "done" when finished.

**What to try:**

1. Start Pi with this extension.
2. Ask: "Read all the TypeScript files in extensions/ and summarize what each one does."
3. Watch Pi create a task list first, then work through each task one by one.
4. The footer will show the task list with live progress indicators.
5. Type `/tilldone` to open an interactive overlay showing all tasks and their statuses.

**Why this matters:** Without structure, an AI can start working, get confused partway
through, and produce incomplete results. This extension forces the AI to plan before
executing — the same principle as writing unit tests before writing production code.

**Analogy:** This is like enforcing a strict "write your pseudocode first" rule before
touching a keyboard. The AI cannot touch any tools until it has written the plan.

---

### Use Case 7: Spawn Background Workers

**Command:**
```bash
just ext-subagent-widget
```

**What it does:** Adds four slash commands that let you spawn and manage background AI
agents. Each background agent gets its own live progress widget displayed above the
editor.

**What to try:**

1. Start Pi with this extension.
2. Type `/sub list all TypeScript files in extensions/ and count lines in each`
3. Watch a new widget appear showing the subagent's ID, status, elapsed time, tool call
   count, and a live preview of its last output line.
4. While the subagent is running, you can continue chatting with the main Pi agent.
5. When the subagent finishes, its result is delivered back to the main agent as a
   follow-up message.
6. Type `/sub summarize the README.md file` to spawn a second subagent. Two widgets now
   show side-by-side progress.
7. Type `/subcont 1 now write a markdown table of the results` to continue the first
   subagent's conversation with a follow-up task.

**Commands added by this extension:**

| Command | What it does |
|---|---|
| `/sub <task>` | Spawn a new background agent with the given task |
| `/subcont <id> <prompt>` | Continue a finished subagent's conversation |
| `/subrm <id>` | Remove a specific subagent widget (kills it if still running) |
| `/subclear` | Clear all subagent widgets |

**Why this matters:** Long-running tasks no longer block your main session. You can
delegate "read and summarize this large codebase" to a background agent while you
continue asking questions in the foreground.

---

### Use Case 8: Multi-Agent Team

**Command:**
```bash
just ext-agent-team
```

**What it does:** Transforms Pi into a **dispatcher**. The primary agent you talk to has
no direct access to the codebase. Instead, it reads your request, selects the right
specialist from a defined team, and delegates the work via the `dispatch_agent` tool.

Above the editor, a grid dashboard shows every agent on the active team with a card for
each:

```
┌──────────────┐  ┌──────────────┐
│ Planner      │  │ Builder      │
│ ● running 4s │  │ ○ idle       │
│ [##---] 20%  │  │ [-----]  0%  │
│ Breaking do… │  │ Waiting...   │
└──────────────┘  └──────────────┘
```

**What to try:**

1. Start Pi with this extension. A team selection dialog appears — choose "default" or
   any available team.
2. Ask: "Read the minimal.ts extension and explain how it works."
3. Watch the dispatcher send this task to the Scout agent (or similar reader agent).
4. The appropriate agent card in the grid shows its status changing from idle to running
   and back to done.
5. The dispatcher summarizes the result for you.

**Commands added by this extension:**

| Command | What it does |
|---|---|
| `/agents-team` | Open a dialog to switch the active team |
| `/agents-list` | Print a list of all loaded agents and their status |
| `/agents-grid <N>` | Set the number of grid columns (1 to 6) |

**Why this matters:** Complex tasks often require multiple specialists. A planner can
break down requirements, a builder can implement them, and a reviewer can check the work —
all coordinated automatically.

---

### Use Case 9: Safety First With Damage Control

**Command:**
```bash
just ext-damage-control
```

**What it does:** Intercepts every bash command and file operation before it executes.
It evaluates the command against a set of rules defined in `.pi/damage-control-rules.yaml`
and either blocks the command outright, asks you to confirm, or allows it through.

**What to try (safe experiments):**

1. Start Pi with this extension.
2. Ask Pi: "Run `rm -rf /tmp/test` to clean up temporary files."
3. The extension will block the command and explain why: `rm` with recursive or force
   flags is on the blocked list.
4. Ask Pi: "Show me the contents of `.env`."
5. The extension blocks this too: `.env` is on the zero-access-paths list.
6. Ask Pi: "List all files in the current directory."
7. This succeeds — `ls` is not a dangerous command.

**Why this matters:** AI agents are powerful but occasionally make mistakes. An agent
asked to "clean up old files" might decide to delete something important. Damage control
acts as a safety net, catching the worst-case scenarios before they happen.

**What the rules cover:**
- Recursive deletion (`rm -rf`, `rmdir --recursive`)
- Force-pushing to git, resetting hard, rewriting history
- Reading sensitive files (`.env`, `~/.ssh/`, `*.pem`, credential files)
- Destructive SQL commands (`DROP TABLE`, `TRUNCATE`, `DELETE` without `WHERE`)
- Cloud CLI destructive operations (AWS, GCP, Firebase, Vercel, Netlify)

---

### Use Case 10: Build Your Own Agent

**Command:**
```bash
just ext-pi-pi
```

**What it does:** Launches a meta-agent — an AI that builds Pi extensions. When you
describe the extension you want, the pi-pi agent spawns a set of parallel expert
subagents, each of which researches a different aspect of the Pi framework documentation
(extension API, TUI components, theme tokens, keybindings, etc.). Their findings are
aggregated and used to write a complete, working extension.

**What to try:**

1. Start Pi with this extension.
2. Type: "Build an extension that shows a live clock in the footer."
3. Watch the expert subagents appear in the console output as they research in parallel.
4. After research completes, the orchestrator writes the full extension code.

**Why this matters:** This is the most advanced pattern in the repository. It demonstrates
recursive AI use: an AI agent that uses other AI agents to improve its own output. The
parallel research pattern also shows how to use multiple specialist subagents to gather
information faster than a single agent could.

---

### Bonus: Stack Extensions Like LEGO

Extensions compose. You can pass multiple `-e` flags to `pi` to load several extensions
at once. The `just open` recipe makes this easy:

```bash
just open minimal tool-counter-widget theme-cycler
```

This opens a new terminal window with three extensions stacked:
- `minimal` — the compact context bar footer
- `tool-counter-widget` — per-tool call counts displayed above the editor
- `theme-cycler` — keyboard shortcuts and `/theme` command for switching themes

You can combine almost any set of extensions. The `justfile` shows the combinations
used most often. To experiment, just swap extension names.

To see every extension running in its own terminal window simultaneously:
```bash
just all
```

---

## 6. Understanding What You See On Screen

Here is a labeled diagram of the Pi terminal interface. Your terminal window is divided
into vertical zones from top to bottom:

```
┌─────────────────────────────────────────────────────────────┐
│  HEADER                                                     │
│  Shows: Pi version, session ID, model name                  │
│  (Some extensions hide or modify this)                      │
├─────────────────────────────────────────────────────────────┤
│  WIDGET AREA (above editor)                                 │
│  Shows: Extension widgets — task lists, agent grids,        │
│  subagent progress bars, purpose banners. Empty by default. │
│                                                             │
│  Example (subagent-widget running):                         │
│  ────────────────────────────────────────────────────────   │
│  ● Subagent #1  list files   (12s)  | Tools: 3             │
│    Scanning extensions/ directory...                        │
│  ────────────────────────────────────────────────────────   │
├─────────────────────────────────────────────────────────────┤
│  RESPONSE AREA (scrollable)                                 │
│  Shows: The conversation history — your prompts and Pi's    │
│  responses. Tool calls (read, bash, grep) appear here as    │
│  collapsible entries showing what the AI did.               │
│                                                             │
│  You:  Write a function that sorts a linked list            │
│                                                             │
│  Pi:   Here is an implementation in C...                    │
│        [read] extensions/minimal.ts  (tool call entry)      │
│        [bash] ls -la  (tool call entry)                     │
│                                                             │
│  (Scroll up with arrow keys or PageUp/PageDown)             │
├─────────────────────────────────────────────────────────────┤
│  INPUT EDITOR                                               │
│  This is where you type. Press Enter to send. Press         │
│  Shift+Enter for a newline within the message. Start a      │
│  line with / to enter a slash command (/theme, /sub, etc.)  │
├─────────────────────────────────────────────────────────────┤
│  FOOTER                                                     │
│  Default: model name, context usage percentage              │
│  With minimal:  modelname  [###-------] 30%                 │
│  With tool-counter: two-line token/cost + tool tally        │
│  (Extensions replace this with custom content)             │
├─────────────────────────────────────────────────────────────┤
│  STATUS LINE                                                │
│  One-line status messages from extensions.                  │
│  Example: "Team: default (3)  |  TillDone: 4 tasks (2 rem)" │
└─────────────────────────────────────────────────────────────┘
```

### Tool Call Entries in the Response Area

When the AI uses a tool (reads a file, runs a bash command, searches with grep), the
tool call appears as an entry in the response area. By default these are shown in a
compact one-line form. Press Ctrl+O (expand tools) to see the full output of each tool call.

### Overlay Dialogs

Some extensions open full-screen overlay dialogs on top of the main UI:
- **Purpose Gate:** text input dialog on startup
- **Team select:** list selection dialog (`/agents-team`)
- **Theme picker:** list selection dialog (`/theme`)
- **TillDone viewer:** task list overlay (`/tilldone`)

Press Escape to close any overlay without taking action.

---

## 7. Commands Cheat Sheet

Slash commands are typed directly in the input editor. They are processed by the
extension that registered them, not sent to the AI.

### Subagent Widget (`just ext-subagent-widget`)

| Command | Description |
|---|---|
| `/sub <task>` | Spawn a new background agent to work on the given task |
| `/subcont <id> <prompt>` | Continue subagent number `<id>` with a follow-up prompt |
| `/subrm <id>` | Remove subagent number `<id>` (kills it if running) |
| `/subclear` | Remove all subagent widgets |

### TillDone (`just ext-tilldone`)

| Command | Description |
|---|---|
| `/tilldone` | Open the task list overlay showing all tasks with statuses |

### Agent Team (`just ext-agent-team`)

| Command | Description |
|---|---|
| `/agents-team` | Open a dialog to switch which agent team is active |
| `/agents-list` | Print all loaded agents, their statuses, and run counts |
| `/agents-grid <N>` | Set grid display column count; N must be between 1 and 6 |

### Agent Chain (`just ext-agent-chain`)

| Command | Description |
|---|---|
| `/chain` | Open a dialog to select and run a defined agent pipeline |
| `/chain-list` | List all available agent chain definitions |

### Theme Cycler (`just ext-theme-cycler`)

| Command | Description |
|---|---|
| `/theme` | Open the theme picker dialog |

### System Select (`just ext-system-select`)

| Command | Description |
|---|---|
| `/system` | Open a picker to load an agent persona as the system prompt |

### Session Replay (`just ext-session-replay`)

| Command | Description |
|---|---|
| `/replay` | Open a scrollable timeline overlay of the session history |

### Pi-Pi Meta-Agent (`just ext-pi-pi`)

| Command | Description |
|---|---|
| `/experts` | Show the list of expert subagents available to the meta-agent |
| `/experts-grid` | Toggle the expert grid dashboard view |

---

## 8. Keyboard Shortcuts

These shortcuts work while the input editor is focused.

### Built-In Pi Shortcuts (always available)

| Shortcut | Action |
|---|---|
| `Enter` | Submit your message to the AI |
| `Shift+Enter` | Insert a newline in your message (for multi-line input) |
| `Escape` | Interrupt the AI while it is responding |
| `Ctrl+D` | Exit Pi |
| `Ctrl+C` | Clear the current input or copy selected text |
| `Shift+Tab` | Cycle thinking level (off → low → medium → high) |
| `Ctrl+P` | Cycle to the next AI model |
| `Ctrl+L` | Open the model selector dialog |
| `Ctrl+O` | Toggle expanded view of tool call results |
| `Ctrl+T` | Toggle thinking mode on or off |
| `Ctrl+G` | Open your `$EDITOR` for multi-line message composition |
| `PageUp` / `PageDown` | Scroll the response area up or down |
| `Up` / `Down` | Navigate response area or select in overlay dialogs |

### Extension-Added Shortcuts

These are added by extensions loaded alongside the base session:

| Shortcut | Extension | Action |
|---|---|---|
| `Ctrl+X` | `theme-cycler` | Cycle to the next theme |
| `Ctrl+Q` | `theme-cycler` | Cycle to the previous theme |

### Overlay-Specific Navigation

When a dialog overlay is open (theme picker, team selector, task viewer):

| Key | Action |
|---|---|
| `Up` / `Down` | Move selection cursor |
| `Enter` | Confirm selection |
| `Escape` | Close without selecting |
| `Ctrl+C` | Close without selecting |

---

## 9. Common Troubleshooting

### "command not found: pi"

Pi is not installed or is not on your `PATH`.

**Fix:**
```bash
npm install -g @mariozechner/pi-coding-agent
```

If you already installed it but still see this error, Node.js's global bin directory
may not be in your `PATH`. Find it with:
```bash
npm bin -g
```
Then add that directory to your `PATH` in `~/.bashrc` or `~/.zshrc`:
```bash
export PATH="$PATH:$(npm bin -g)"
```

---

### "command not found: just"

The `just` task runner is not installed.

**Fix (macOS):**
```bash
brew install just
```

**Fix (Linux with Cargo):**
```bash
cargo install just
```

**Fix (download binary):**
Visit https://github.com/casey/just/releases and download the binary for your platform.

---

### "command not found: bun"

Bun is not installed or not on your `PATH`.

**Fix:**
```bash
curl -fsSL https://bun.sh/install | bash
```

Then close and reopen your terminal (the installer modifies your shell profile but the
change only takes effect in a new session).

---

### Pi starts but says "No API key" or refuses to respond

Your API key is not loaded into the environment.

**Fix (quickest):** Run Pi through `just`, which auto-loads `.env`:
```bash
just pi
just ext-minimal
```

**Fix (manual):** Source the `.env` file before running Pi:
```bash
source .env && pi
```

**Check your keys are set:**
```bash
echo $ANTHROPIC_API_KEY    # Should print your key, not empty
echo $OPENAI_API_KEY       # Should print your key, not empty
```

**Make sure `.env` exists and has a real key in it:**
```bash
cat .env
# Should show lines like: ANTHROPIC_API_KEY=sk-ant-abc123...
# NOT: ANTHROPIC_API_KEY=sk-ant-...   (the placeholder from .env.sample)
```

---

### "Extension not loading" or Pi starts without the expected widget

The extension file path may be wrong, or the file may have a TypeScript syntax error.

**Check the file exists:**
```bash
ls extensions/
```
All extension files should be listed. Each ends in `.ts`.

**Run directly to see errors:**
```bash
pi -e extensions/minimal.ts
```

If there is a syntax error in the extension, Pi will print the error message. Check the
line number it references.

**Make sure you typed the extension name correctly.** Extension names are case-sensitive
on Linux/macOS. `Minimal.ts` is not the same as `minimal.ts`.

---

### The `just open` command does not open a new terminal window

The `just open` recipe uses `osascript` (macOS's scripting bridge) to open a new
Terminal window. This only works on macOS with the Terminal app.

On Linux or Windows, use the command directly in a new terminal:
```bash
pi -e extensions/minimal.ts -e extensions/theme-cycler.ts
```

---

### Pi is responding very slowly

This is usually caused by a slow AI provider, not a local problem.

- Try switching to a faster model with `Ctrl+L` (model selector).
- Check your internet connection.
- Some models (especially "thinking" models) are intentionally slower because they
  reason through problems before answering. Toggle thinking off with `Ctrl+T`.

---

### The context bar is filling up quickly

The AI's context window fills up as conversation history grows. When it reaches 100%,
Pi will automatically compact (summarize) the oldest messages to free up space.

To start fresh manually, exit Pi (Ctrl+D) and start a new session.

---

## 10. What's Next?

Once you have worked through the ten use cases above, you are ready to go deeper.

### Read These First

| Document | What it covers |
|---|---|
| `README.md` | Overview, extension table, usage patterns, orchestration patterns |
| `COMPARISON.md` | Feature-by-feature comparison of Claude Code vs Pi Agent |
| `THEME.md` | Color token reference for writing custom extensions |
| `TOOLS.md` | Built-in tool function signatures available inside extensions |
| `RESERVED_KEYS.md` | Which keyboard shortcuts Pi owns and which are free for extensions |

### Understand the Extension Code

The simplest extension to read first is `extensions/minimal.ts`. It is about 30 lines
and demonstrates the core pattern: listen for `session_start`, then call `ctx.ui.setFooter()`
with a render function. Once you understand that pattern, the other extensions are
variations on the same theme.

Read the extensions in this order for a smooth learning curve:

1. `extensions/minimal.ts` — footer customization
2. `extensions/pure-focus.ts` — removing UI elements
3. `extensions/purpose-gate.ts` — intercepting user input and adding a widget
4. `extensions/tool-counter.ts` — reading session history and live tool tracking
5. `extensions/subagent-widget.ts` — spawning child processes and managing widgets
6. `extensions/tilldone.ts` — registering custom tools with the AI
7. `extensions/agent-team.ts` — full multi-agent orchestration

### Experiment With Extensions

Try modifying an existing extension. For example:
- Change the bar character in `minimal.ts` from `#` to `█` for a solid block style.
- Add a new rule to `.pi/damage-control-rules.yaml` for a command you want blocked.
- Add a new agent by creating a `.md` file in `.pi/agents/` following the existing format.

### Pi Documentation

For full API reference and advanced topics:

| Doc | URL |
|---|---|
| Extension system API | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md |
| TypeScript SDK reference | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/sdk.md |
| Providers and models | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/providers.md |
| Settings reference | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/settings.md |
| Skills (shareable agent behaviors) | https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/skills.md |

### Learn Agentic Coding Patterns

For broader patterns in agentic software development beyond this specific project:

- [Tactical Agentic Coding course](https://agenticengineer.com/tactical-agentic-coding?y=pivscc)
- [IndyDevDan YouTube channel](https://www.youtube.com/@indydevdan)
- [Mario Zechner on Twitter/X](https://x.com/badlogicgames) — creator of Pi Coding Agent
