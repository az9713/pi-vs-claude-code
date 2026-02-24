# Tutorial 7: Dynamic System Prompts — Shaping AI Behavior

**Prerequisites:** Tutorials 1–6, or familiarity with registering commands and handling
`session_start`. You should be comfortable reading a 30–50 line extension.

**Navigation:** [← Tutorial 6](./06-input-interception.md) | [Tutorial 8 →](./08-background-subagent.md)

---

## What You Will Build

A `persona` extension that lets you define AI personas as plain TypeScript objects and
switch between them at runtime. When you select a persona, two things happen:

1. Its custom instructions are prepended to the system prompt before every LLM call.
2. Its allowed tool list restricts what the AI can actually do.

You will end up with four personas:

| Persona | Restricted to | Behavior |
|---|---|---|
| **Default** | All tools | Normal Pi behavior |
| **Careful Reviewer** | `read, grep, find, ls` | Reads code and reports problems; refuses to write |
| **Speed Builder** | All tools | Implementation-focused; skips lengthy explanations |
| **Security Auditor** | `read, bash, grep, find, ls` | Hunts for security issues and explains them |

Run the extension:

```bash
pi -e extensions/persona.ts
```

Then type `/persona` to open the picker.

---

## Background: What Is a System Prompt?

When you send a message to Pi, what the LLM actually receives is not just your message.
It receives a carefully structured packet of text:

```
[System prompt]         ← Instructions the LLM follows for the whole session
[Conversation history]  ← Every previous turn, alternating user/assistant
[Your new message]      ← What you just typed
```

The **system prompt** is the most important piece. It tells the model who it is, what
tools it has, what rules to follow, and how to format responses. Pi generates a default
system prompt automatically. Extensions can intercept and modify it.

Think of it this way: if the conversation history is your program's heap — runtime data
that grows as the session proceeds — then the system prompt is a global constant that
shapes the entire execution environment.

---

## What You Will Learn

- `pi.on("before_agent_start")` — the hook that fires before every LLM call
- Returning a new `systemPrompt` from `before_agent_start` to change AI behavior
- `pi.getActiveTools()` and `pi.setActiveTools()` for tool restriction
- `ctx.ui.select()` for dropdown dialogs
- `ctx.ui.setStatus()` for persistent status line messages
- How tool restriction physically limits what the AI can do (not just asks it nicely)

---

## Step 1: Create the File and Define Personas

Create `extensions/persona.ts`. Start with the imports and the persona data:

```typescript
/**
 * Persona — Switch AI behavior via /persona command
 *
 * Injects a persona's instructions into the system prompt before every LLM
 * call and restricts tools to the persona's allowed set.
 *
 * Usage: pi -e extensions/persona.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
```

Now define the persona type and the persona list. In TypeScript, an `interface` is like
a struct in C or a class with only public fields in Java — it describes the shape of an
object without any behavior:

```typescript
interface Persona {
  name: string;
  description: string;
  systemPromptSnippet: string;
  tools: string[] | null; // null means "use all default tools"
}

const PERSONAS: Persona[] = [
  {
    name: "Default",
    description: "Normal Pi behavior — no modifications",
    systemPromptSnippet: "",
    tools: null,
  },
  {
    name: "Careful Reviewer",
    description: "Read-only — reviews code for bugs and issues without modifying anything",
    systemPromptSnippet: `You are a careful code reviewer. Your role is to read code and identify
bugs, design issues, unclear logic, and potential improvements. You do NOT
write or modify code — you only read and report. When asked to write code,
explain instead what the implementation should do and why. Be thorough and
precise in your analysis.`,
    tools: ["read", "grep", "find", "ls"],
  },
  {
    name: "Speed Builder",
    description: "Implementation-focused — builds fast with minimal explanation",
    systemPromptSnippet: `You are a fast, implementation-focused engineer. Get things done quickly
and efficiently. Skip lengthy explanations unless asked. Write working code
immediately. Prefer direct action over discussion. When you complete a task,
give a one-sentence summary of what you did, not a paragraph.`,
    tools: null, // all tools available — builders need everything
  },
  {
    name: "Security Auditor",
    description: "Security-focused — hunts for vulnerabilities and explains them",
    systemPromptSnippet: `You are a security auditor. Your job is to find security vulnerabilities,
misconfigurations, and unsafe patterns in code. Look for: injection attacks,
authentication bypasses, insecure defaults, hardcoded secrets, missing input
validation, path traversal vulnerabilities, and privilege escalation. You can
run bash commands to probe the system but MUST NOT modify any files. Report
findings with severity ratings (Critical, High, Medium, Low) and explain
the attack vector for each issue.`,
    tools: ["read", "bash", "grep", "find", "ls"],
  },
];
```

**Why an array of objects rather than a map?** Because order matters for display —
we want "Default" always first in the picker — and because arrays are simpler to
iterate in TypeScript.

---

## Step 2: Write the Extension Function

All Pi extension logic lives inside a single default-exported function. This is a fixed
contract: Pi loads your `.ts` file and calls whatever you export as `default`:

```typescript
export default function (pi: ExtensionAPI) {
  let activePersona: Persona = PERSONAS[0]; // start with Default
  let defaultTools: string[] = [];          // filled in on session_start
```

The two module-level variables use **closure** — they are captured by all the event
handlers and command handlers you register below. Closure in JavaScript/TypeScript works
exactly like a lambda capturing local variables in Java or a functor capturing state in C++.
The variables live as long as any function that references them does.

---

## Step 3: Capture Default Tools on Session Start

You need to save the default tool list before changing it, so you can restore it when
the user picks "Default":

```typescript
  pi.on("session_start", async (_event, ctx) => {
    // Save the full tool list before any persona modifies it
    defaultTools = pi.getActiveTools();

    // Reset to Default persona on every new session
    activePersona = PERSONAS[0];
    ctx.ui.setStatus("persona", "Persona: Default");
  });
```

`pi.getActiveTools()` returns an array of tool name strings like
`["read", "write", "edit", "bash", "grep", "find", "ls"]`. Store this now, because
once you call `pi.setActiveTools(["read"])`, the original list is gone.

`ctx.ui.setStatus("persona", "Persona: Default")` adds a named entry to the status
line at the bottom of the screen. The first argument is a **key** — it uniquely
identifies this status entry so you can update it later without adding duplicates. The
second argument is the display string.

---

## Step 4: Register the /persona Command

```typescript
  pi.registerCommand("persona", {
    description: "Switch AI persona",
    handler: async (_args, ctx) => {
      // Build option strings for the picker dialog
      const options = PERSONAS.map(
        (p) => `${p.name} — ${p.description}`
      );

      // ctx.ui.select() opens a full-screen list picker and returns the chosen string
      // Returns undefined if the user presses Escape without choosing
      const choice = await ctx.ui.select("Select Persona", options);
      if (choice === undefined) return; // user cancelled

      // Find which persona was chosen by matching back to the index
      const index = options.indexOf(choice);
      activePersona = PERSONAS[index];

      // Apply tool restriction
      if (activePersona.tools !== null) {
        pi.setActiveTools(activePersona.tools);
      } else {
        pi.setActiveTools(defaultTools);
      }

      // Update the status line
      ctx.ui.setStatus("persona", `Persona: ${activePersona.name}`);

      // Show a brief confirmation popup
      ctx.ui.notify(
        `Switched to: ${activePersona.name}\n${activePersona.description}`,
        "success"
      );
    },
  });
```

`ctx.ui.select()` is an `async` function that suspends until the user makes a selection
or presses Escape. In JavaScript, `await` on an async function is like calling a
blocking method in Java — execution pauses here and resumes when the Promise resolves.
The difference is that the rest of the event loop keeps running, so Pi's UI stays
responsive while the dialog is open.

---

## Step 5: Intercept Every LLM Call

This is the core mechanism. `before_agent_start` fires every time Pi is about to send
a new request to the LLM — before your message is processed, before any tool calls in
the response. Returning a new `systemPrompt` replaces the default:

```typescript
  pi.on("before_agent_start", async (event, _ctx) => {
    // Default persona has no snippet — don't modify anything
    if (!activePersona.systemPromptSnippet) return;

    // Prepend the persona's instructions to Pi's default system prompt.
    // This is the concatenation pattern: persona first, then Pi's defaults.
    // The LLM reads top-to-bottom, so earlier instructions carry more weight.
    return {
      systemPrompt: activePersona.systemPromptSnippet + "\n\n" + event.systemPrompt,
    };
  });
} // end of export default function
```

**Critical detail:** `event.systemPrompt` is Pi's automatically generated system
prompt for this turn. It contains the tool descriptions, the codebase context, and
Pi's built-in rules. If you return just your persona snippet without concatenating
`event.systemPrompt`, the AI loses all knowledge of its tools and will be unable to
use them. Always concatenate.

The pattern `myContent + "\n\n" + event.systemPrompt` puts your instructions first.
LLMs give more weight to content that appears early in the system prompt. If you want
your additions to feel like guidelines rather than overrides, put them at the end instead:
`event.systemPrompt + "\n\n" + myContent`.

---

## Full Final Code

Putting it all together — approximately 80 lines:

```typescript
/**
 * Persona — Switch AI behavior via /persona command
 *
 * Usage: pi -e extensions/persona.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

interface Persona {
  name: string;
  description: string;
  systemPromptSnippet: string;
  tools: string[] | null;
}

const PERSONAS: Persona[] = [
  {
    name: "Default",
    description: "Normal Pi behavior — no modifications",
    systemPromptSnippet: "",
    tools: null,
  },
  {
    name: "Careful Reviewer",
    description: "Read-only — reviews code for bugs and issues without modifying anything",
    systemPromptSnippet: `You are a careful code reviewer. Your role is to read code and identify
bugs, design issues, unclear logic, and potential improvements. You do NOT
write or modify code — you only read and report. When asked to write code,
explain instead what the implementation should do and why. Be thorough and
precise in your analysis.`,
    tools: ["read", "grep", "find", "ls"],
  },
  {
    name: "Speed Builder",
    description: "Implementation-focused — builds fast with minimal explanation",
    systemPromptSnippet: `You are a fast, implementation-focused engineer. Get things done quickly
and efficiently. Skip lengthy explanations unless asked. Write working code
immediately. Prefer direct action over discussion. When you complete a task,
give a one-sentence summary of what you did, not a paragraph.`,
    tools: null,
  },
  {
    name: "Security Auditor",
    description: "Security-focused — hunts for vulnerabilities and explains them",
    systemPromptSnippet: `You are a security auditor. Your job is to find security vulnerabilities,
misconfigurations, and unsafe patterns in code. Look for: injection attacks,
authentication bypasses, insecure defaults, hardcoded secrets, missing input
validation, path traversal vulnerabilities, and privilege escalation. You can
run bash commands to probe the system but MUST NOT modify any files. Report
findings with severity ratings (Critical, High, Medium, Low) and explain
the attack vector for each issue.`,
    tools: ["read", "bash", "grep", "find", "ls"],
  },
];

export default function (pi: ExtensionAPI) {
  let activePersona: Persona = PERSONAS[0];
  let defaultTools: string[] = [];

  pi.on("session_start", async (_event, ctx) => {
    defaultTools = pi.getActiveTools();
    activePersona = PERSONAS[0];
    ctx.ui.setStatus("persona", "Persona: Default");
  });

  pi.registerCommand("persona", {
    description: "Switch AI persona",
    handler: async (_args, ctx) => {
      const options = PERSONAS.map((p) => `${p.name} — ${p.description}`);
      const choice = await ctx.ui.select("Select Persona", options);
      if (choice === undefined) return;

      const index = options.indexOf(choice);
      activePersona = PERSONAS[index];

      if (activePersona.tools !== null) {
        pi.setActiveTools(activePersona.tools);
      } else {
        pi.setActiveTools(defaultTools);
      }

      ctx.ui.setStatus("persona", `Persona: ${activePersona.name}`);
      ctx.ui.notify(
        `Switched to: ${activePersona.name}\n${activePersona.description}`,
        "success"
      );
    },
  });

  pi.on("before_agent_start", async (event, _ctx) => {
    if (!activePersona.systemPromptSnippet) return;
    return {
      systemPrompt: activePersona.systemPromptSnippet + "\n\n" + event.systemPrompt,
    };
  });
}
```

---

## Testing It

Run the extension:

```bash
pi -e extensions/persona.ts
```

**Test 1 — Verify Default behavior:**
Ask: `Write a simple hello-world function in TypeScript.`
Pi should write the code immediately.

**Test 2 — Switch to Careful Reviewer:**
Type `/persona`, select "Careful Reviewer".
Now ask: `Write a simple hello-world function in TypeScript.`
The AI should *refuse* or *explain* rather than write. This is the tool restriction
working: the AI physically cannot call `write` or `edit`, so even if it wanted to write
code, it has no mechanism to do so.

**Test 3 — Verify tool restriction in the UI:**
With Careful Reviewer active, check the Pi status or look at the footer. If you have
`minimal.ts` loaded alongside, you can observe which tools are listed.

**Test 4 — Security Auditor:**
Type `/persona`, select "Security Auditor".
Ask: `Review this project for security issues.`
Watch the auditor probe the codebase. Note that it has `bash` access (for running
security-relevant commands) but not `write` or `edit` (it cannot make changes).

**Test 5 — Restore Default:**
Type `/persona`, select "Default".
The status line should update, and the full tool set should be restored.

---

## Deep Dive: System Prompt Architecture

### When Does `before_agent_start` Fire?

`before_agent_start` fires once per LLM call, which is not the same as once per user
message. A single user message can trigger multiple LLM calls if the AI uses tools and
then reasons about their results. The sequence looks like this:

```
User sends message
  → before_agent_start fires → LLM call 1
  → LLM decides to call "read" tool
  → tool executes
  → before_agent_start fires → LLM call 2
  → LLM produces final text response
```

This means your system prompt injection runs on every turn of the inner agent loop,
not just on the first one. That is usually what you want — the persona should persist
for the entire interaction, not just the opening call.

### Replacing vs. Augmenting

There are three patterns:

**Pattern A — Augment at the top (strongest influence):**
```typescript
return { systemPrompt: mySnippet + "\n\n" + event.systemPrompt };
```
Your instructions are read first. The LLM follows them more strictly.

**Pattern B — Augment at the bottom (softer influence):**
```typescript
return { systemPrompt: event.systemPrompt + "\n\n" + mySnippet };
```
Pi's built-in instructions take precedence. Your additions feel like addenda.

**Pattern C — Full replacement (dangerous):**
```typescript
return { systemPrompt: mySnippet }; // ignores event.systemPrompt entirely
```
The LLM loses all knowledge of its tools, the codebase, and Pi's rules. Avoid this
unless you are intentionally building a completely custom agent from scratch.

The `purpose-gate.ts` extension uses Pattern B (appending to the end). The `agent-team.ts`
extension uses Pattern C intentionally — the dispatcher agent is a completely different
character that should not follow Pi's normal codebase instructions.

### Tool Restriction: Soft vs. Hard Limits

Injecting a system prompt that says "do not write files" is a **soft limit** — you are
asking the AI politely. A determined or confused LLM might still try to write.

Calling `pi.setActiveTools(["read"])` is a **hard limit** — the `write` and `edit`
tools are physically not registered in this session. The LLM cannot see them in the
tool catalog, and even if it hallucinated a call to `write`, the framework would reject
it. Hard limits are more reliable than soft instructions.

Combining both — system prompt instructions AND tool restriction — is the most robust
approach. The system prompt helps the LLM understand *why* it cannot do something, and
the tool restriction prevents it from trying anyway.

---

## Exercises

**Exercise 1 — Add a fifth persona.**
Create a "Documentation Writer" persona with tools `["read", "write", "grep", "find", "ls"]`
and instructions focused on writing clear, accurate markdown documentation.

**Exercise 2 — Persist the active persona across sessions.**
Use Node.js's `fs.writeFileSync()` to save the active persona name to a file
(e.g., `.pi/active-persona.txt`) in `session_start`, and read it back on startup to
restore the last-used persona.

**Exercise 3 — Add a persona-specific footer.**
In the persona command handler, call `ctx.ui.setFooter()` to display the active persona
name in the footer bar, color-coded by type (green for builder, red for auditor, etc.).

**Exercise 4 — Dynamic persona from a file.**
Instead of hardcoding personas in the TypeScript array, read them from `.pi/personas/`
directory as markdown files with frontmatter (similar to how `system-select.ts` reads
agent definitions). This makes personas user-configurable without editing TypeScript.

---

## Summary

You have learned:

- `before_agent_start` fires before every LLM call and can return a modified `systemPrompt`
- Always concatenate with `event.systemPrompt` to preserve Pi's tool descriptions
- `pi.getActiveTools()` returns the current tool list; save it before modifying
- `pi.setActiveTools(["read", "bash"])` hard-restricts the available tools
- `ctx.ui.select()` shows a blocking dropdown picker; `undefined` means Escape
- `ctx.ui.setStatus("key", "text")` manages named status line entries

The complete production version of this pattern lives in `extensions/system-select.ts`,
which loads personas from markdown files in multiple directories rather than hardcoding them.

---

**Navigation:** [← Tutorial 6](./06-input-interception.md) | [Tutorial 8 →](./08-background-subagent.md)
