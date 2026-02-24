# Tutorial 5: Event Interception — Building a Safety Gate

**Prerequisites:** Tutorial 4, or comfort with `pi.registerTool()` and session event
handlers. You should understand how event handlers are registered and what `ctx` contains.

**What You Will Build:** A `safe-bash` extension that watches every tool call the AI
makes, classifies bash commands as dangerous or medium-risk, blocks or confirms them, and
maintains a running audit log with a live status count.

**What You Will Learn:**
- `pi.on("tool_call")` — the event that fires before any tool executes
- `isToolCallEventType()` for type-safe narrowing of the event
- Returning `{ block: true, reason: "..." }` to prevent tool execution
- `ctx.ui.confirm()` for interactive yes/no prompts
- `ctx.abort()` to stop the AI's current turn after blocking
- `pi.appendEntry()` for structured audit logging to the session
- `ctx.ui.setStatus()` for a live status line counter
- Regular expressions for pattern matching against command strings

---

## Background: The Tool Call Event Pipeline

Before writing code, understand the sequence of events when the AI decides to run a tool:

```
LLM emits tool call JSON
        |
        v
Pi fires "tool_call" event to ALL registered handlers (in registration order)
        |
        v
Each handler returns { block: true, reason } OR { block: false }
        |
        v
If ANY handler returned block:true → tool does NOT run
    → the reason string is sent back to the LLM as a simulated tool error
    → the LLM reads the reason and decides what to do next (usually apologizes
      and tries a different approach)
        |
        v
If all handlers returned block:false → tool executes normally
```

This pipeline is how `damage-control.ts` works in production. It registers one
`tool_call` handler that checks the command against a YAML rule set. Your safe-bash
extension will do the same thing, but inline and simplified.

The key insight: **you cannot stop a tool call from within the tool itself**. You must
register a `tool_call` event handler at the extension level. The handler fires before the
tool runs, and returning `{ block: true }` is the only way to prevent execution.

---

## Step 1: Create the File and Imports

Create `extensions/safe-bash.ts`:

```typescript
/**
 * Safe-Bash Extension — Intercepts bash tool calls and blocks dangerous commands.
 *
 * Dangerous commands (rm -rf, sudo, etc.) are blocked outright.
 * Medium-risk commands (git push, npm publish) prompt the user for confirmation.
 * All blocked commands are logged to the session via pi.appendEntry().
 *
 * Usage: pi -e extensions/safe-bash.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
```

`isToolCallEventType` is a type guard function provided by Pi. Without it, the `event`
object in a `tool_call` handler is a generic type that does not expose tool-specific
fields like `event.input.command`. Calling `isToolCallEventType("bash", event)` narrows
the type and gives you access to the bash-specific `input.command` field.

---

## Step 2: Define the Pattern Rules

Below the imports, define the rule set. In TypeScript, an array of objects with a
consistent shape is the idiomatic replacement for a `struct[]` or `List<Rule>` in C/Java:

```typescript
// ── Rule definitions ────────────────────────────────────────────────────────
//
// Each rule has:
//   pattern — a regular expression matched against the full command string
//   reason  — the message sent back to the LLM (and shown to the user) when matched
//   ask     — if true, prompt the user for confirmation instead of auto-blocking

interface SafeRule {
    pattern: RegExp;
    reason: string;
    ask?: boolean; // optional — defaults to undefined (falsy)
}

const RULES: SafeRule[] = [
    // ── Hard blocks — always blocked, no prompt ──────────────────────────
    {
        pattern: /rm\s+-[a-z]*r[a-z]*f|rm\s+-[a-z]*f[a-z]*r/i,
        reason: "Recursive force deletion (rm -rf) is not permitted.",
    },
    {
        pattern: /\bsudo\b/,
        reason: "Sudo commands are not permitted in this session.",
    },
    {
        pattern: /\bchmod\s+777\b/,
        reason: "chmod 777 (world-writable) is not permitted.",
    },
    {
        pattern: />\s*\/dev\/sd[a-z]|>\s*\/dev\/nvme/i,
        reason: "Writing directly to block devices is not permitted.",
    },
    {
        pattern: /\bdd\b.*\bof=\/dev\//i,
        reason: "Writing to block devices via dd is not permitted.",
    },
    {
        pattern: /:(){ :|:& };:/,
        reason: "Fork bomb detected.",
    },

    // ── Medium risk — ask the user before allowing ───────────────────────
    {
        pattern: /\bgit\s+push\b/,
        reason: "git push will send changes to a remote repository.",
        ask: true,
    },
    {
        pattern: /\bnpm\s+publish\b|\bbun\s+publish\b/,
        reason: "Publishing a package to the npm registry is irreversible.",
        ask: true,
    },
    {
        pattern: /\bgit\s+(push\s+--force|push\s+-f)\b/,
        reason: "Force push will rewrite remote history.",
        ask: true,
    },
    {
        pattern: /\bcurl\b.*\|\s*(ba)?sh/i,
        reason: "Piping curl output directly into a shell is a security risk.",
        ask: true,
    },
    {
        pattern: /\bwget\b.*\|\s*(ba)?sh/i,
        reason: "Piping wget output directly into a shell is a security risk.",
        ask: true,
    },
];
```

### Why `RegExp` as a value

In TypeScript (and JavaScript), regular expression literals look like `/pattern/flags`.
They are a built-in type — `RegExp` — with a `.test(string)` method that returns `true`
if the pattern matches anywhere in the string.

The `/i` flag makes the match case-insensitive. The `\b` metacharacter matches a word
boundary (the transition between a word character and a non-word character), so `\bsudo\b`
matches the standalone word `sudo` but not `pseudocode`.

The `interface SafeRule` definition gives the array a typed shape. The `ask?: boolean`
means the field is optional — `undefined` when absent, which is falsy in a boolean
context.

---

## Step 3: The Extension Entry Point and State

```typescript
// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {
    let blockedCount = 0;
    let allowedCount = 0;
```

Two counters live in closure scope and are updated by the `tool_call` handler.

---

## Step 4: The Session Start Handler

```typescript
    // ── Session init ────────────────────────────────────────────────────────

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);

        // Set an initial status line entry so the user can see the extension is active.
        ctx.ui.setStatus("Safe-bash: 0 blocked / 0 allowed", "safe-bash");

        ctx.ui.notify("Safe-bash active: dangerous commands will be blocked.");
    });
```

`ctx.ui.setStatus(text, key)` sets a named status line entry. The `key` string identifies
this entry so future calls with the same key update it in place rather than appending a
new entry.

`ctx.ui.notify(text)` shows a transient notification message in the TUI.

---

## Step 5: The Tool Call Interceptor

This is the core of the extension. Add it after the session start handler:

```typescript
    // ── Tool call interceptor ───────────────────────────────────────────────

    pi.on("tool_call", async (event, ctx) => {
        // isToolCallEventType narrows the event type. Without this check, the
        // TypeScript compiler does not know that event.input has a .command field.
        // Conceptually: if (event.toolName !== "bash") return { block: false };
        // But isToolCallEventType also narrows the TYPE so we get autocomplete.
        if (!isToolCallEventType("bash", event)) {
            return { block: false };
        }

        const command = event.input.command;

        // Test the command against each rule in order. The first match wins.
        for (const rule of RULES) {
            if (!rule.pattern.test(command)) continue;

            // We have a match. Decide whether to hard-block or ask.
            if (rule.ask) {
                // Medium-risk: ask the user whether to allow this command.
                const confirmed = await ctx.ui.confirm(
                    "Safe-Bash: Confirm Command",
                    `This command may have significant side effects:\n\n` +
                    `  ${command}\n\n` +
                    `Reason: ${rule.reason}\n\n` +
                    `Do you want to allow it?`,
                    { timeout: 60000 }, // Wait up to 60 seconds for a response
                );

                if (confirmed) {
                    allowedCount++;
                    updateStatus(ctx);
                    pi.appendEntry("safe-bash-log", {
                        command,
                        rule: rule.reason,
                        action: "confirmed_by_user",
                        timestamp: new Date().toISOString(),
                    });
                    return { block: false };
                } else {
                    // User denied the medium-risk command — treat as a hard block.
                    blockedCount++;
                    updateStatus(ctx);
                    pi.appendEntry("safe-bash-log", {
                        command,
                        rule: rule.reason,
                        action: "denied_by_user",
                        timestamp: new Date().toISOString(),
                    });
                    // abort() stops the AI's current turn immediately.
                    // Without it, the LLM receives the block reason and may try
                    // an alternative approach that also gets blocked. abort() gives
                    // control back to the user instead.
                    ctx.abort();
                    return {
                        block: true,
                        reason: `BLOCKED by safe-bash: ${rule.reason} (denied by user). Do not attempt alternative approaches that achieve the same result. Tell the user what was blocked and ask how to proceed.`,
                    };
                }
            } else {
                // Hard block: no confirmation, always denied.
                blockedCount++;
                updateStatus(ctx);
                ctx.ui.notify(`Blocked: ${rule.reason}`);
                pi.appendEntry("safe-bash-log", {
                    command,
                    rule: rule.reason,
                    action: "blocked",
                    timestamp: new Date().toISOString(),
                });
                ctx.abort();
                return {
                    block: true,
                    reason: `BLOCKED by safe-bash: ${rule.reason}. Do not attempt alternative approaches that achieve the same result. Tell the user what was blocked and ask how to proceed.`,
                };
            }
        }

        // No rule matched — allow the command through.
        allowedCount++;
        updateStatus(ctx);
        return { block: false };
    });
```

### Key API calls explained

**`isToolCallEventType("bash", event)`**

This is a TypeScript type guard. It checks whether the tool call event is for the `bash`
tool. If it is, TypeScript narrows the type of `event` so that `event.input` is known to
have a `command: string` field. Without this call, accessing `event.input.command` would
be a type error because the generic tool call event type does not include tool-specific
input fields.

Available tool names for `isToolCallEventType`: `"bash"`, `"read"`, `"write"`, `"edit"`,
`"grep"`, `"find"`, `"ls"`.

**`ctx.ui.confirm(title, body, options)`**

Opens a yes/no dialog in the TUI. The `await` is essential: the function returns a
Promise that resolves to `true` (yes) or `false` (no / timeout). The `timeout` option
specifies how many milliseconds to wait before automatically returning `false`.

This is an async operation even though it looks synchronous in the code — the JavaScript
runtime processes other events while waiting for the user's keypress. The `async` keyword
on the handler function is what makes `await` usable here.

**`ctx.abort()`**

Stops the AI's current turn. Without `ctx.abort()`, after receiving the block reason, the
LLM would immediately try again with a different command. That usually leads to a cascade
of blocks. `abort()` hands control back to the user, who can then decide what to tell the
AI next.

Compare this to `damage-control.ts`, which also calls `ctx.abort()` on every block.

**`pi.appendEntry(entryType, data)`**

Appends a custom entry to the session log. The first argument is a string tag (your
choice). The second argument is any JSON-serializable object. These entries appear in the
session tree but are not sent to the LLM. They are useful for audit logs, debugging, and
custom session replay views.

---

## Step 6: The Status Update Helper

Add the `updateStatus` helper inside the main function body, before the event handlers:

```typescript
    // ── Helpers ─────────────────────────────────────────────────────────────

    const updateStatus = (ctx: ExtensionContext) => {
        ctx.ui.setStatus(
            `Safe-bash: ${blockedCount} blocked / ${allowedCount} allowed`,
            "safe-bash",
        );
    };
```

Wait — `ExtensionContext` is referenced here but we did not import it in Step 1. Add it:

```typescript
import type { ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";
```

The `const updateStatus = (ctx: ExtensionContext) => { ... }` syntax defines a function
using an arrow function and assigns it to a `const` variable. This is equivalent to:

```typescript
function updateStatus(ctx: ExtensionContext): void { ... }
```

Both forms work. The arrow function form is used here because it captures the outer scope
(the `blockedCount` and `allowedCount` variables) naturally. In Java terms, it is like
a lambda that closes over the enclosing class's fields.

---

## Step 7: Run and Test

```bash
pi -e extensions/safe-bash.ts
```

Test the hard block. Ask the AI:

```
Please run: rm -rf /tmp/test-directory
```

The AI will attempt a bash call. Before it executes, safe-bash intercepts it, matches
the `rm -rf` pattern, logs the block, updates the status line, and returns
`{ block: true, reason: "..." }`. The AI receives the reason and should respond by
explaining it was blocked.

Test the confirmation dialog. Ask:

```
Please push the current branch to origin.
```

The AI will call `git push`. You will see a confirm dialog appear. Press `n` to deny or
`y` to allow. The status counter will update either way.

Test that safe commands pass through:

```
Please list the files in the current directory.
```

The `ls` command matches no rule and passes through normally.

---

## Complete File

```typescript
/**
 * Safe-Bash Extension — Intercepts bash tool calls and blocks dangerous commands.
 *
 * Usage: pi -e extensions/safe-bash.ts
 */

import type { ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";

// ── Rule definitions ────────────────────────────────────────────────────────

interface SafeRule {
    pattern: RegExp;
    reason: string;
    ask?: boolean;
}

const RULES: SafeRule[] = [
    // Hard blocks
    {
        pattern: /rm\s+-[a-z]*r[a-z]*f|rm\s+-[a-z]*f[a-z]*r/i,
        reason: "Recursive force deletion (rm -rf) is not permitted.",
    },
    {
        pattern: /\bsudo\b/,
        reason: "Sudo commands are not permitted in this session.",
    },
    {
        pattern: /\bchmod\s+777\b/,
        reason: "chmod 777 (world-writable) is not permitted.",
    },
    {
        pattern: />\s*\/dev\/sd[a-z]|>\s*\/dev\/nvme/i,
        reason: "Writing directly to block devices is not permitted.",
    },
    {
        pattern: /\bdd\b.*\bof=\/dev\//i,
        reason: "Writing to block devices via dd is not permitted.",
    },
    {
        pattern: /:(){ :|:& };:/,
        reason: "Fork bomb detected.",
    },

    // Medium risk — confirm before allowing
    {
        pattern: /\bgit\s+push\b/,
        reason: "git push will send changes to a remote repository.",
        ask: true,
    },
    {
        pattern: /\bnpm\s+publish\b|\bbun\s+publish\b/,
        reason: "Publishing a package to the npm registry is irreversible.",
        ask: true,
    },
    {
        pattern: /\bgit\s+(push\s+--force|push\s+-f)\b/,
        reason: "Force push will rewrite remote history.",
        ask: true,
    },
    {
        pattern: /\bcurl\b.*\|\s*(ba)?sh/i,
        reason: "Piping curl output directly into a shell is a security risk.",
        ask: true,
    },
    {
        pattern: /\bwget\b.*\|\s*(ba)?sh/i,
        reason: "Piping wget output directly into a shell is a security risk.",
        ask: true,
    },
];

// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {
    let blockedCount = 0;
    let allowedCount = 0;

    const updateStatus = (ctx: ExtensionContext) => {
        ctx.ui.setStatus(
            `Safe-bash: ${blockedCount} blocked / ${allowedCount} allowed`,
            "safe-bash",
        );
    };

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        ctx.ui.setStatus("Safe-bash: 0 blocked / 0 allowed", "safe-bash");
        ctx.ui.notify("Safe-bash active: dangerous commands will be blocked.");
    });

    pi.on("tool_call", async (event, ctx) => {
        if (!isToolCallEventType("bash", event)) {
            return { block: false };
        }

        const command = event.input.command;

        for (const rule of RULES) {
            if (!rule.pattern.test(command)) continue;

            if (rule.ask) {
                const confirmed = await ctx.ui.confirm(
                    "Safe-Bash: Confirm Command",
                    `This command may have significant side effects:\n\n` +
                    `  ${command}\n\n` +
                    `Reason: ${rule.reason}\n\n` +
                    `Do you want to allow it?`,
                    { timeout: 60000 },
                );

                if (confirmed) {
                    allowedCount++;
                    updateStatus(ctx);
                    pi.appendEntry("safe-bash-log", {
                        command,
                        rule: rule.reason,
                        action: "confirmed_by_user",
                        timestamp: new Date().toISOString(),
                    });
                    return { block: false };
                } else {
                    blockedCount++;
                    updateStatus(ctx);
                    pi.appendEntry("safe-bash-log", {
                        command,
                        rule: rule.reason,
                        action: "denied_by_user",
                        timestamp: new Date().toISOString(),
                    });
                    ctx.abort();
                    return {
                        block: true,
                        reason: `BLOCKED by safe-bash: ${rule.reason} (denied by user). Do not attempt alternative approaches. Tell the user what was blocked and ask how to proceed.`,
                    };
                }
            } else {
                blockedCount++;
                updateStatus(ctx);
                ctx.ui.notify(`Blocked: ${rule.reason}`);
                pi.appendEntry("safe-bash-log", {
                    command,
                    rule: rule.reason,
                    action: "blocked",
                    timestamp: new Date().toISOString(),
                });
                ctx.abort();
                return {
                    block: true,
                    reason: `BLOCKED by safe-bash: ${rule.reason}. Do not attempt alternative approaches. Tell the user what was blocked and ask how to proceed.`,
                };
            }
        }

        allowedCount++;
        updateStatus(ctx);
        return { block: false };
    });
}
```

---

## Deep Dive: The Event Pipeline in Detail

When the LLM decides to call a tool, this is the exact sequence inside Pi:

1. Pi parses the tool call JSON from the LLM's response stream.
2. Pi fires a `tool_call` event to every registered handler. **Handlers fire in
   registration order** — first the handler from the extension loaded first, then the
   second, and so on. If you stack two extensions both registering `tool_call` handlers,
   both fire.
3. Each handler returns either `{ block: false }` (allow) or `{ block: true, reason }`.
4. Pi collects all results. If **any** handler returned `block: true`, the tool does not
   execute.
5. Pi sends the `reason` string back to the LLM as a simulated tool error result. The LLM
   treats this as if the tool ran and returned an error.
6. The LLM reads the reason and responds. Usually it apologizes and tries a different
   approach — unless the reason contains explicit instructions like "Do not attempt
   alternative approaches."

This is exactly how `damage-control.ts` works. It loads a YAML file of rules on startup,
then registers one `tool_call` handler that checks every tool call (not just bash) against
those rules. The pattern is the same; only the rule set is different.

The `block: true` return value does not raise an exception or crash the AI. It is a
normal, expected result. The LLM is designed to handle tool errors gracefully. Your reason
string is the primary communication channel — write it clearly.

---

## Exercises

**Exercise 1 — Block writes to sensitive paths**

Add a second event handler that intercepts `write` and `edit` tool calls. Use
`isToolCallEventType("write", event)` and `isToolCallEventType("edit", event)`. Block any
call where `event.input.path` includes `.env`, `.ssh`, or `credentials`. Return a block
with an appropriate reason. Verify by asking the AI to edit your `.env` file.

**Exercise 2 — Add a `/safe-bash-log` command**

Use `pi.registerCommand("safe-bash-log", { ... })` (see the commands pattern in
`tilldone.ts`). In the handler, call `ctx.sessionManager.getBranch()` and filter for
entries where `entry.type === "customEntry"` and `entry.entryType === "safe-bash-log"`.
Print each blocked command and its reason using `ctx.ui.notify()`.

**Exercise 3 — Path-based blocking with RegExp**

Add a rule that blocks bash commands containing paths ending in `.pem`, `.key`, or `.p12`
(private key files). The pattern needs to match these extensions anywhere in the command
string: `/\.(pem|key|p12)\b/i`. Test by asking the AI to read or cat a file with one of
those extensions.

---

## What's Next

- **Tutorial 6: Overlay UI** — build an interactive full-screen overlay with keyboard
  navigation, triggered by a `/help` slash command.
- Study `extensions/damage-control.ts` — this tutorial's safe-bash is a simplified
  version. `damage-control.ts` loads rules from a YAML file, handles path-based access
  control for read/write/edit tools as well as bash, and covers 61 patterns. Every API
  call used there is now familiar from this tutorial.
