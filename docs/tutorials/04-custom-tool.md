# Tutorial 4: Registering a Custom Tool

**Prerequisites:** Tutorials 01–03, or familiarity with `extensions/minimal.ts` and the
session-start / footer pattern. You should understand what `pi.on()` does and what
`ExtensionContext` is.

**What You Will Build:** A `notes` tool that the AI can call to save, list, retrieve, and
remove text notes during a session. Notes survive branch switching because their state is
reconstructed from the session history on every session event.

**What You Will Learn:**
- `pi.registerTool()` — the full API surface
- TypeBox schemas (`Type.Object`, `Type.String`, `Type.Number`, `Type.Optional`,
  `StringEnum`)
- The `execute` function signature and every parameter it receives
- The tool result format: `{ content: [{type: "text", text: "..."}], details: {...} }`
- Custom `renderCall` and `renderResult` for styled display
- The `details` field: how to persist tool state into the session log
- State reconstruction from session history (the pattern used by `tilldone.ts`)

---

## Background: What Is a Tool, Exactly?

Before writing code, understand what Pi does with a registered tool:

1. **Discovery:** When your extension calls `pi.registerTool(...)`, Pi stores the tool's
   name, description, and parameter schema.

2. **System prompt injection:** When a new agent turn begins, Pi converts all registered
   tools into the format the LLM expects (a JSON schema block). This is embedded in the
   system prompt automatically. You do not write any prompt text for this.

3. **LLM decision:** The LLM reads the tool description and decides — on its own — when
   to call the tool. A well-written description is the single biggest factor in whether
   the AI uses your tool correctly.

4. **Execution:** When the LLM emits a tool call, Pi validates the arguments against your
   TypeBox schema, then calls your `execute` function.

5. **Result:** Your `execute` function returns a result object. Pi feeds it back to the
   LLM as the tool's output, and the LLM continues its response.

Think of registering a tool as publishing a function signature + docs to a very literal
programmer who will call it exactly as documented. If the description is vague, the
programmer will misuse it. If the description is precise, they will use it perfectly.

---

## Step 1: Create the File and Set Up Imports

Create `extensions/notes.ts` with the following skeleton:

```typescript
/**
 * Notes Tool — Lets the AI save, list, retrieve, and remove notes during a session.
 *
 * State is stored in each tool result's `details` field and reconstructed from the
 * session branch on session_start / session_switch / session_fork.
 *
 * Usage: pi -e extensions/notes.ts
 */

import { StringEnum } from "@mariozechner/pi-ai";
import type { ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";
import { Text } from "@mariozechner/pi-tui";
import { Type } from "@sinclair/typebox";
import { applyExtensionDefaults } from "./themeMap.ts";
```

### Import-by-import explanation

`StringEnum` from `@mariozechner/pi-ai`
: A helper that creates a TypeBox schema for a string that must be one of a fixed set of
  values — exactly like an `enum` in Java or C. This is not in the standard TypeBox
  library; Pi provides it. You pass an array of string literals as a `const` tuple.

`ExtensionAPI, ExtensionContext` from `@mariozechner/pi-coding-agent`
: The two main interfaces. `ExtensionAPI` (called `pi` by convention) is what you receive
  in the default export function. `ExtensionContext` (called `ctx`) is what you receive
  inside event handlers. They are imported as `type` because they are only needed for
  TypeScript's compile-time type checking — the actual objects are provided by Pi at
  runtime.

`Text` from `@mariozechner/pi-tui`
: Pi's terminal UI text node. `renderCall` and `renderResult` must return a TUI component.
  `Text` is the simplest one — it just holds a string, possibly with ANSI color codes.

`Type` from `@sinclair/typebox`
: The TypeBox library for building JSON Schema objects. Every tool parameter set is
  defined with this.

`applyExtensionDefaults` from `./themeMap.ts`
: The project-wide helper that sets the theme and terminal title on session start.

---

## Step 2: Define the Data Model

Below the imports, define the TypeScript interface for a note and the TypeBox parameter
schema for the tool:

```typescript
// ── Data model ─────────────────────────────────────────────────────────────

interface Note {
    id: number;       // Auto-incrementing, starts at 1
    text: string;     // The note content
    createdAt: string; // ISO 8601 timestamp, e.g. "2025-01-15T14:30:00.000Z"
}

interface NotesDetails {
    action: string;
    notes: Note[];
    nextId: number;
    error?: string;
}
```

`interface` in TypeScript is like a `struct` in C or a POJO interface in Java. It defines
the shape of an object without any implementation. The `?` on `error` means the field is
optional — it may or may not be present.

`NotesDetails` is the shape of what we store in the tool result's `details` field. We
need `notes` and `nextId` to fully reconstruct state.

Now define the TypeBox schema for the tool's parameters:

```typescript
// ── Parameter schema ────────────────────────────────────────────────────────

const NotesParams = Type.Object({
    action: StringEnum(["add", "list", "get", "remove"] as const),
    text: Type.Optional(
        Type.String({ description: "Note text — required for 'add'" })
    ),
    id: Type.Optional(
        Type.Number({ description: "Note ID — required for 'get' and 'remove'" })
    ),
});
```

### TypeBox concepts

`Type.Object({...})`
: Creates a schema for a JSON object with named fields. This is the top-level wrapper for
  all tool parameter schemas.

`StringEnum(["add", "list", "get", "remove"] as const)`
: Creates a schema that accepts only one of those four exact strings. The `as const`
  annotation is TypeScript syntax that tells the compiler to treat the array as a tuple of
  literal string types rather than a general `string[]`. Without it, TypeScript would
  widen the type and `StringEnum` would not be able to infer the literal union.

`Type.Optional(Type.String(...))`
: Wraps a schema in `Optional`, meaning the field may be absent entirely. The object
  passed to `Type.String()` is a description that Pi includes in the tool documentation
  given to the LLM.

`Type.Number(...)`
: A schema for a JSON number. Note IDs are numbers, not strings.

---

## Step 3: The Extension Entry Point and State Variables

```typescript
// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {
    // In-memory state — reconstructed from session history on every session event
    let notes: Note[] = [];
    let nextId = 1;
```

This is the standard Pi extension pattern. The `export default function` is the entry
point Pi calls once at startup. Everything inside it is closure scope — the `notes` array
and `nextId` counter live for the lifetime of the extension but are not global (they are
captured by every closure defined inside this function).

In Java terms: imagine a class with private fields `notes` and `nextId`, and every
registered handler is a method of that class. The closure is the same concept.

---

## Step 4: State Reconstruction Helper

Add the following helper function inside the main function body (before the tool
registration):

```typescript
    // ── State reconstruction ─────────────────────────────────────────────────
    //
    // Pi sessions are trees. When the user forks, switches branches, or replays
    // history, our in-memory state must be rebuilt by replaying the tool results
    // stored on the current branch.
    //
    // Every time our tool runs, it stores a snapshot of all notes in result.details.
    // Replaying the branch means finding the LAST notes tool result and copying
    // its snapshot into our live variables.

    const reconstructState = (ctx: ExtensionContext) => {
        notes = [];
        nextId = 1;

        for (const entry of ctx.sessionManager.getBranch()) {
            if (entry.type !== "message") continue;
            const msg = entry.message;
            if (msg.role !== "toolResult" || msg.toolName !== "notes") continue;

            const details = msg.details as NotesDetails | undefined;
            if (details) {
                notes = details.notes;
                nextId = details.nextId;
            }
        }
    };
```

### Why this pattern exists

The session is stored as a tree of message entries. When you fork a branch and come back
to the original branch, the in-memory `notes` array still reflects the fork's state.
Re-reading the branch entries and replaying the last tool result snapshot is the reliable
way to synchronize in-memory state with the session's actual history.

`ctx.sessionManager.getBranch()` returns an array of all entries on the current branch,
in chronological order.

Each entry has a `type` field. Entries where `type === "message"` contain actual
conversation messages. Within those, `msg.role` tells you whether it is a user message,
an assistant message, or a tool result.

We look for entries where `role === "toolResult"` and `toolName === "notes"` — these are
the results our tool previously returned. The `details` field contains our snapshot. We
overwrite `notes` and `nextId` with each snapshot we find, so after the loop, we have the
most recent snapshot.

---

## Step 5: Register Session Events

```typescript
    // ── Session events ────────────────────────────────────────────────────────

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        reconstructState(ctx);
    });

    pi.on("session_switch", async (_event, ctx) => reconstructState(ctx));
    pi.on("session_fork",   async (_event, ctx) => reconstructState(ctx));
    pi.on("session_tree",   async (_event, ctx) => reconstructState(ctx));
```

`session_start` fires when Pi starts a new session. We apply the theme and reconstruct
state (which will be empty on a fresh session, but this handles a session loaded from
disk).

`session_switch`, `session_fork`, and `session_tree` fire when the user navigates the
session tree. Each one needs to reconstruct state because the current branch has changed.

The `_event` prefix convention means "I accept this parameter but do not use it." This is
idiomatic TypeScript/JavaScript — the underscore signals intent to future readers.

---

## Step 6: Register the Tool

Now the main event. Add the tool registration after the session event handlers:

```typescript
    // ── Tool registration ─────────────────────────────────────────────────────

    pi.registerTool({
        // The name the LLM uses to call this tool. Must be unique across all
        // registered tools. Use lowercase with hyphens.
        name: "notes",

        // A short label shown in Pi's UI when the tool is called or returned.
        label: "Notes",

        // The description the LLM reads to decide when and how to call this tool.
        // This is the most important field. Be explicit about what each action
        // requires. Vague descriptions produce incorrect tool calls.
        description:
            "Manage persistent notes for this session. " +
            "Actions: " +
            "'add' (requires text) — save a new note, returns the note ID; " +
            "'list' — returns all notes with their IDs and timestamps; " +
            "'get' (requires id) — retrieves one specific note; " +
            "'remove' (requires id) — deletes a note permanently. " +
            "Use this tool to remember important information across a long session.",

        // The TypeBox schema for this tool's parameters.
        parameters: NotesParams,
```

### Every field in `pi.registerTool()`

`name`
: The string the LLM sends when it wants to call your tool. It appears in the system
  prompt alongside the description and schema. Keep it short and verb-noun or noun format.

`label`
: A human-readable display label used in Pi's TUI. It is not sent to the LLM.

`description`
: The prose the LLM reads to understand your tool. Write it as if explaining to a
  meticulous but literal programmer. Include what each action does, what parameters it
  requires, and what it returns. The LLM will follow this description precisely.

`parameters`
: The TypeBox schema object. Pi uses this to validate incoming arguments and to generate
  the JSON Schema sent to the LLM.

---

## Step 7: Implement the Execute Function

The `execute` function is where your tool's logic lives. Add it inside the
`pi.registerTool({...})` call:

```typescript
        // The execute function is called by Pi whenever the LLM uses your tool.
        //
        // Parameters:
        //   toolCallId — a unique ID for this specific call (useful for logging)
        //   params     — the validated, typed arguments from the LLM
        //   signal     — an AbortSignal; check signal.aborted for long operations
        //   onUpdate   — call this to stream partial results before the final return
        //   ctx        — the ExtensionContext for this session
        async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
            // A snapshot helper — captures the current state for the details field.
            // We call this in every return so the state is always up to date.
            const makeDetails = (action: string, error?: string): NotesDetails => ({
                action,
                notes: [...notes], // spread copies the array — avoids reference sharing
                nextId,
                ...(error ? { error } : {}),
            });

            switch (params.action) {
                case "add": {
                    if (!params.text) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'text' is required for action 'add'" }],
                            details: makeDetails("add", "text required"),
                        };
                    }

                    const note: Note = {
                        id: nextId++,
                        text: params.text,
                        createdAt: new Date().toISOString(),
                    };
                    notes.push(note);

                    ctx.ui.setStatus(`Notes: ${notes.length} saved`, "notes");

                    return {
                        content: [{ type: "text" as const, text: `Note #${note.id} saved: "${note.text}"` }],
                        details: makeDetails("add"),
                    };
                }

                case "list": {
                    if (notes.length === 0) {
                        return {
                            content: [{ type: "text" as const, text: "No notes saved yet." }],
                            details: makeDetails("list"),
                        };
                    }

                    const lines = notes.map(
                        (n) => `#${n.id} [${n.createdAt}]: ${n.text}`
                    );
                    return {
                        content: [{ type: "text" as const, text: lines.join("\n") }],
                        details: makeDetails("list"),
                    };
                }

                case "get": {
                    if (params.id === undefined) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'id' is required for action 'get'" }],
                            details: makeDetails("get", "id required"),
                        };
                    }
                    const found = notes.find((n) => n.id === params.id);
                    if (!found) {
                        return {
                            content: [{ type: "text" as const, text: `Note #${params.id} not found.` }],
                            details: makeDetails("get", `#${params.id} not found`),
                        };
                    }
                    return {
                        content: [{ type: "text" as const, text: `#${found.id}: ${found.text}` }],
                        details: makeDetails("get"),
                    };
                }

                case "remove": {
                    if (params.id === undefined) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'id' is required for action 'remove'" }],
                            details: makeDetails("remove", "id required"),
                        };
                    }
                    const idx = notes.findIndex((n) => n.id === params.id);
                    if (idx === -1) {
                        return {
                            content: [{ type: "text" as const, text: `Note #${params.id} not found.` }],
                            details: makeDetails("remove", `#${params.id} not found`),
                        };
                    }
                    const removed = notes.splice(idx, 1)[0];
                    ctx.ui.setStatus(`Notes: ${notes.length} saved`, "notes");
                    return {
                        content: [{ type: "text" as const, text: `Removed note #${removed.id}: "${removed.text}"` }],
                        details: makeDetails("remove"),
                    };
                }

                default:
                    return {
                        content: [{ type: "text" as const, text: `Unknown action: ${params.action}` }],
                        details: makeDetails("list", `unknown action: ${params.action}`),
                    };
            }
        },
```

### Return format: content and details

Every tool result must have a `content` array. This is what the LLM receives back. The
simplest entry is `{ type: "text", text: "..." }`. The `as const` annotation on the
string `"text"` is a TypeScript requirement: without it the type would be inferred as
`string`, which is too broad for the discriminated union the API expects.

The `details` field is not sent to the LLM — it is stored in the session log as metadata
attached to the tool result entry. When we reconstruct state from session history, we read
`details` back. This is how state survives branch switching.

`notes.splice(idx, 1)` removes one element at position `idx` and returns it as a
single-element array. We immediately destructure with `[0]` to get the removed element.
This is equivalent to `list.remove(idx)` in Java.

---

## Step 8: Add `renderCall` and `renderResult`

These are optional but greatly improve readability. Add them inside `pi.registerTool({})`,
after the `execute` function:

```typescript
        // renderCall: formats how the tool call appears in Pi's response area.
        // Called when the LLM is about to run the tool.
        // Returns a TUI component — Text is the simplest.
        renderCall(args, theme) {
            // Build a line like: "notes add  "save the current API endpoint""
            let display = theme.fg("toolTitle", theme.bold("notes ")) +
                          theme.fg("muted", args.action);

            if (args.text) {
                display += " " + theme.fg("dim", `"${args.text}"`);
            }
            if (args.id !== undefined) {
                display += " " + theme.fg("accent", `#${args.id}`);
            }

            return new Text(display, 0, 0);
        },

        // renderResult: formats how the tool result appears in Pi's response area.
        // Called after execute() returns.
        // The second parameter is { expanded: boolean } — expanded is true when the user
        // has pressed Ctrl+O to expand tool call results.
        renderResult(result, _renderCtx, theme) {
            const details = result.details as NotesDetails | undefined;

            // If no details (should not happen), fall back to raw text
            if (!details) {
                const text = result.content[0];
                return new Text(text?.type === "text" ? text.text : "", 0, 0);
            }

            // Show error in red
            if (details.error) {
                return new Text(theme.fg("error", `Error: ${details.error}`), 0, 0);
            }

            switch (details.action) {
                case "add": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(
                        theme.fg("success", "+ ") + theme.fg("muted", text),
                        0, 0
                    );
                }
                case "list": {
                    const count = details.notes.length;
                    return new Text(
                        theme.fg("accent", `${count} note${count === 1 ? "" : "s"}`),
                        0, 0
                    );
                }
                case "get": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(theme.fg("muted", text), 0, 0);
                }
                case "remove": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(
                        theme.fg("warning", "- ") + theme.fg("dim", text),
                        0, 0
                    );
                }
                default:
                    return new Text(theme.fg("dim", "done"), 0, 0);
            }
        },
    }); // end pi.registerTool
} // end export default function
```

### Text constructor

`new Text(content, paddingLeft, paddingRight)` — the second and third arguments are
padding. For tool renderers, `0, 0` is standard.

### Theme tokens

Pi themes expose these semantic color tokens, usable with `theme.fg("token", text)`:

| Token | Typical use |
|---|---|
| `"toolTitle"` | Tool name in call display |
| `"accent"` | Highlighted values, IDs |
| `"success"` | Confirmations, add operations |
| `"warning"` | Removals, cautions |
| `"error"` | Errors |
| `"muted"` | Secondary information |
| `"dim"` | De-emphasized text |

`theme.bold(text)` wraps text in bold ANSI codes.

---

## Step 9: Run and Test

```bash
pi -e extensions/notes.ts
```

Or add it to your justfile and use:
```bash
just ext-notes
```

Once Pi is running, ask the AI something that would benefit from notes:

```
Please save a note: "The API base URL is https://api.example.com/v2"
```

The AI will call `notes add` with the text. You will see the styled call and result in
the response area.

Try follow-up prompts:
```
List all my saved notes.
Retrieve note #1.
Remove note #1.
```

---

## Complete File

Here is the complete `extensions/notes.ts`:

```typescript
/**
 * Notes Tool — Lets the AI save, list, retrieve, and remove notes during a session.
 *
 * State is stored in each tool result's `details` field and reconstructed from the
 * session branch on session_start / session_switch / session_fork.
 *
 * Usage: pi -e extensions/notes.ts
 */

import { StringEnum } from "@mariozechner/pi-ai";
import type { ExtensionAPI, ExtensionContext } from "@mariozechner/pi-coding-agent";
import { Text } from "@mariozechner/pi-tui";
import { Type } from "@sinclair/typebox";
import { applyExtensionDefaults } from "./themeMap.ts";

// ── Data model ──────────────────────────────────────────────────────────────

interface Note {
    id: number;
    text: string;
    createdAt: string;
}

interface NotesDetails {
    action: string;
    notes: Note[];
    nextId: number;
    error?: string;
}

// ── Parameter schema ────────────────────────────────────────────────────────

const NotesParams = Type.Object({
    action: StringEnum(["add", "list", "get", "remove"] as const),
    text: Type.Optional(
        Type.String({ description: "Note text — required for 'add'" })
    ),
    id: Type.Optional(
        Type.Number({ description: "Note ID — required for 'get' and 'remove'" })
    ),
});

// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {
    let notes: Note[] = [];
    let nextId = 1;

    // ── State reconstruction ────────────────────────────────────────────────

    const reconstructState = (ctx: ExtensionContext) => {
        notes = [];
        nextId = 1;

        for (const entry of ctx.sessionManager.getBranch()) {
            if (entry.type !== "message") continue;
            const msg = entry.message;
            if (msg.role !== "toolResult" || msg.toolName !== "notes") continue;

            const details = msg.details as NotesDetails | undefined;
            if (details) {
                notes = details.notes;
                nextId = details.nextId;
            }
        }
    };

    // ── Session events ──────────────────────────────────────────────────────

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        reconstructState(ctx);
    });

    pi.on("session_switch", async (_event, ctx) => reconstructState(ctx));
    pi.on("session_fork",   async (_event, ctx) => reconstructState(ctx));
    pi.on("session_tree",   async (_event, ctx) => reconstructState(ctx));

    // ── Tool registration ───────────────────────────────────────────────────

    pi.registerTool({
        name: "notes",
        label: "Notes",
        description:
            "Manage persistent notes for this session. " +
            "Actions: " +
            "'add' (requires text) — save a new note, returns the note ID; " +
            "'list' — returns all notes with their IDs and timestamps; " +
            "'get' (requires id) — retrieves one specific note; " +
            "'remove' (requires id) — deletes a note permanently. " +
            "Use this tool to remember important information across a long session.",
        parameters: NotesParams,

        async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
            const makeDetails = (action: string, error?: string): NotesDetails => ({
                action,
                notes: [...notes],
                nextId,
                ...(error ? { error } : {}),
            });

            switch (params.action) {
                case "add": {
                    if (!params.text) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'text' is required for action 'add'" }],
                            details: makeDetails("add", "text required"),
                        };
                    }
                    const note: Note = {
                        id: nextId++,
                        text: params.text,
                        createdAt: new Date().toISOString(),
                    };
                    notes.push(note);
                    ctx.ui.setStatus(`Notes: ${notes.length} saved`, "notes");
                    return {
                        content: [{ type: "text" as const, text: `Note #${note.id} saved: "${note.text}"` }],
                        details: makeDetails("add"),
                    };
                }

                case "list": {
                    if (notes.length === 0) {
                        return {
                            content: [{ type: "text" as const, text: "No notes saved yet." }],
                            details: makeDetails("list"),
                        };
                    }
                    const lines = notes.map(
                        (n) => `#${n.id} [${n.createdAt}]: ${n.text}`
                    );
                    return {
                        content: [{ type: "text" as const, text: lines.join("\n") }],
                        details: makeDetails("list"),
                    };
                }

                case "get": {
                    if (params.id === undefined) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'id' is required for action 'get'" }],
                            details: makeDetails("get", "id required"),
                        };
                    }
                    const found = notes.find((n) => n.id === params.id);
                    if (!found) {
                        return {
                            content: [{ type: "text" as const, text: `Note #${params.id} not found.` }],
                            details: makeDetails("get", `#${params.id} not found`),
                        };
                    }
                    return {
                        content: [{ type: "text" as const, text: `#${found.id}: ${found.text}` }],
                        details: makeDetails("get"),
                    };
                }

                case "remove": {
                    if (params.id === undefined) {
                        return {
                            content: [{ type: "text" as const, text: "Error: 'id' is required for action 'remove'" }],
                            details: makeDetails("remove", "id required"),
                        };
                    }
                    const idx = notes.findIndex((n) => n.id === params.id);
                    if (idx === -1) {
                        return {
                            content: [{ type: "text" as const, text: `Note #${params.id} not found.` }],
                            details: makeDetails("remove", `#${params.id} not found`),
                        };
                    }
                    const removed = notes.splice(idx, 1)[0];
                    ctx.ui.setStatus(`Notes: ${notes.length} saved`, "notes");
                    return {
                        content: [{ type: "text" as const, text: `Removed note #${removed.id}: "${removed.text}"` }],
                        details: makeDetails("remove"),
                    };
                }

                default:
                    return {
                        content: [{ type: "text" as const, text: `Unknown action: ${params.action}` }],
                        details: makeDetails("list", `unknown action: ${params.action}`),
                    };
            }
        },

        renderCall(args, theme) {
            let display = theme.fg("toolTitle", theme.bold("notes ")) +
                          theme.fg("muted", args.action);
            if (args.text) {
                display += " " + theme.fg("dim", `"${args.text}"`);
            }
            if (args.id !== undefined) {
                display += " " + theme.fg("accent", `#${args.id}`);
            }
            return new Text(display, 0, 0);
        },

        renderResult(result, _renderCtx, theme) {
            const details = result.details as NotesDetails | undefined;
            if (!details) {
                const text = result.content[0];
                return new Text(text?.type === "text" ? text.text : "", 0, 0);
            }
            if (details.error) {
                return new Text(theme.fg("error", `Error: ${details.error}`), 0, 0);
            }
            switch (details.action) {
                case "add": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(theme.fg("success", "+ ") + theme.fg("muted", text), 0, 0);
                }
                case "list": {
                    const count = details.notes.length;
                    return new Text(
                        theme.fg("accent", `${count} note${count === 1 ? "" : "s"}`),
                        0, 0
                    );
                }
                case "get": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(theme.fg("muted", text), 0, 0);
                }
                case "remove": {
                    const msg = result.content[0];
                    const text = msg?.type === "text" ? msg.text : "";
                    return new Text(theme.fg("warning", "- ") + theme.fg("dim", text), 0, 0);
                }
                default:
                    return new Text(theme.fg("dim", "done"), 0, 0);
            }
        },
    });
}
```

---

## Deep Dive: How the AI Knows About Your Tool

When your extension calls `pi.registerTool(...)`, Pi stores the tool internally. When a
new agent turn begins, Pi serializes every registered tool into the format required by the
LLM's API — a JSON block that includes the tool's name, description, and parameter schema.
This block is included in the system prompt the LLM receives at the start of every turn.

The LLM reads this documentation and decides, based solely on the description and the
user's request, whether to call the tool. It constructs the arguments itself. Pi receives
the raw JSON arguments, validates them against your TypeBox schema, and only calls your
`execute` function if validation passes.

This has one practical consequence: **the description is the interface contract**. The
LLM will call your tool exactly as documented. If you say `add (requires text)`, the LLM
will include `text` when calling `add`. If you say `remove (requires id)`, the LLM will
include `id` when calling `remove`. The clearer your description, the more reliably the
LLM uses your tool.

Contrast this with `renderCall` and `renderResult`, which are only for human display.
The LLM never sees these.

---

## Exercises

**Exercise 1 — Add a search action**

Add a fifth action `"search"` to `StringEnum` and to the schema. It takes an optional
`query: Type.String(...)` parameter. In the `execute` switch, implement a case that
filters `notes` by whether `note.text.toLowerCase().includes(params.query.toLowerCase())`
and returns the matching notes formatted as a list.

**Exercise 2 — Display timestamps in renderResult**

Modify the `"list"` case in `renderResult` to show each note on its own line with a
formatted timestamp. Create a helper `formatDate(iso: string): string` that converts an
ISO 8601 string like `"2025-01-15T14:30:00.000Z"` to something like `"Jan 15, 14:30"`.
The `Text` component accepts newlines in its content string.

**Exercise 3 — Gate other tools on having at least one note**

Add a `tool_call` event handler like the one in `tilldone.ts`. If the calling tool is not
`"notes"` and `notes.length === 0`, return `{ block: true, reason: "..." }` with a
message telling the AI to use the notes tool to save at least one note before working.
This teaches the gate pattern used by `tilldone.ts` without the full complexity of that
extension.

---

## What's Next

- **Tutorial 5: Event Interception** — build a safety gate that intercepts bash tool calls
  and blocks dangerous commands before they run.
- **Tutorial 6: Overlay UI** — build an interactive full-screen overlay with keyboard
  navigation, triggered by a slash command.
- Study `extensions/tilldone.ts` — this tutorial's notes tool is a simplified version.
  `tilldone.ts` adds a blocking gate, auto-nudge on agent completion, a footer widget, a
  widget above the editor, and an interactive overlay. Every concept used there is now
  familiar.
