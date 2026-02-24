# Tutorial 3: Slash Commands and Keyboard Shortcuts

**Series:** Pi Extension Tutorials
**Difficulty:** Intermediate
**Time:** 40–55 minutes
**Prerequisite:** [Tutorial 2: Building a Live Widget](./02-custom-widget.md)

---

## What You Will Build

A `bookmarks` extension with five capabilities:

- `/bookmark add <name>` — saves the current working directory with a timestamp
- `/bookmark list` — displays all saved bookmarks in a notification popup
- `/bookmark go <name>` — demonstrates the selection pattern by looking up a bookmark
- `/bookmark remove <name>` — deletes a saved bookmark
- `Ctrl+B` — opens an interactive picker to choose and inspect a bookmark

This extension keeps everything in memory using a `Map`. An exercise at the end shows how
to persist bookmarks across sessions using `pi.appendEntry()`.

---

## What You Will Learn

- `pi.registerCommand()` — how to add slash commands with argument parsing
- `pi.registerShortcut()` — how to bind `Ctrl+Key` combinations
- `ctx.ui.notify()` — showing brief popup notifications with severity levels
- `ctx.ui.select()` — opening an interactive list picker and reading the selection
- `ctx.cwd` — reading the agent's current working directory
- Extension state management with a `Map`
- Argument parsing patterns for sub-commands

---

## How Slash Commands Work

When a user types `/bookmark add my-project` in the input editor, Pi intercepts the input
before it reaches the AI model. Pi looks for a registered command named `bookmark`, and if
found, calls its handler with the string `"add my-project"` as the `args` parameter.

The handler is responsible for parsing `args` — Pi gives you the raw string after the
command name and you split it however you need.

Commands are registered with `pi.registerCommand(name, { description, handler })`. The
`name` must not include the `/` prefix. Commands are available from the moment the
extension loads — they do not require waiting for `session_start`.

---

## How Keyboard Shortcuts Work

`pi.registerShortcut("ctrl+b", { description, handler })` registers a keyboard shortcut.
Pi intercepts the key combination before it reaches the input editor. The handler receives
the `ExtensionContext` (`ctx`) as its only argument.

Shortcuts are also registered at extension load time, not inside `session_start`. See
`RESERVED_KEYS.md` in the project root for a list of keys that Pi uses internally (avoid
those).

---

## Step 1: Create the File

```bash
touch extensions/bookmarks.ts
```

---

## Step 2: Define the Data Structure

Open `extensions/bookmarks.ts` and add:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";

interface Bookmark {
    path: string;
    time: Date;
}

export default function (pi: ExtensionAPI) {
    const bookmarks = new Map<string, Bookmark>();
}
```

### `interface Bookmark`

In TypeScript, an `interface` declares the shape of an object — which properties it has
and what types they are. Think of it as a Java interface or a C struct definition:

```typescript
// TypeScript interface
interface Bookmark {
    path: string;   // full filesystem path
    time: Date;     // JavaScript Date object
}
```

This is type information only. It creates no runtime code. The TypeScript compiler uses
it to check that objects you create actually have the declared shape.

### `new Map<string, Bookmark>()`

JavaScript's `Map` is a key-value store with ordered insertion. Unlike plain objects
(`{}`), `Map` correctly handles any type of key, preserves insertion order, and has a
proper size property. This is equivalent to Java's `LinkedHashMap<String, Bookmark>`.

`Map<string, Bookmark>` is the TypeScript generic type annotation — it tells the compiler
what types the keys and values are. At runtime this annotation is erased; only `new Map()`
is executed.

Useful `Map` methods you will use:
- `map.set(key, value)` — insert or update
- `map.get(key)` — look up by key, returns `undefined` if missing
- `map.has(key)` — check whether a key exists (returns boolean)
- `map.delete(key)` — remove a key–value pair (returns boolean: `true` if deleted)
- `map.size` — number of entries
- `for (const [key, value] of map)` — iterate in insertion order

---

## Step 3: Register the session_start Handler

Add the session lifecycle handler:

```typescript
export default function (pi: ExtensionAPI) {
    const bookmarks = new Map<string, Bookmark>();

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        bookmarks.clear();
        ctx.ui.notify("Bookmarks extension loaded. Use /bookmark add <name>.", "info");
    });
}
```

`bookmarks.clear()` resets the in-memory store each time a new session starts. This
ensures a fresh slate — bookmarks do not carry over between sessions. (Exercise 2 shows
how to persist them if you want them to survive restarts.)

`ctx.ui.notify(message, severity)` shows a brief notification popup. The severity can be
`"info"`, `"success"`, `"warning"`, or `"error"`, which controls the color. The popup
appears briefly and fades, so it is appropriate for feedback messages, not critical
information.

---

## Step 4: Register the /bookmark Command

Now the core of the tutorial. Add the command registration after the `session_start`
handler (but still inside the exported function):

```typescript
pi.registerCommand("bookmark", {
    description: "Manage bookmarks: /bookmark add <name> | list | go <name> | remove <name>",
    handler: async (args, ctx) => {
        const trimmed = (args ?? "").trim();
        const spaceIdx = trimmed.indexOf(" ");

        // Split into sub-command and remainder
        const subCmd = spaceIdx === -1 ? trimmed : trimmed.slice(0, spaceIdx);
        const rest   = spaceIdx === -1 ? ""      : trimmed.slice(spaceIdx + 1).trim();

        switch (subCmd) {
            case "add":    return handleAdd(rest, ctx);
            case "list":   return handleList(ctx);
            case "go":     return handleGo(rest, ctx);
            case "remove": return handleRemove(rest, ctx);
            default:
                ctx.ui.notify(
                    `Unknown sub-command: "${subCmd}". ` +
                    `Use: add <name> | list | go <name> | remove <name>`,
                    "warning"
                );
        }
    },
});
```

### `args ?? ""`

`??` is the nullish coalescing operator. It returns the right side if the left side is
`null` or `undefined`, and the left side otherwise. This is stricter than `||`: the
expression `"" || "default"` returns `"default"` (because `""` is falsy), but
`"" ?? "default"` returns `""` (because `""` is not null or undefined).

We use `??` here because an empty args string is valid (the user typed `/bookmark` with
no arguments), and we want to preserve it as `""` rather than substituting a fallback.

### Argument parsing

The raw `args` string is everything after the command name. For `/bookmark add my-project`,
`args` is `"add my-project"`.

We split by finding the first space with `indexOf(" ")`. Everything before the space is
the sub-command (`"add"`), everything after is the remainder (`"my-project"`). If there is
no space, the entire string is the sub-command and the remainder is empty.

This two-level parsing (command → sub-command → remaining text) is a common pattern in
CLI tools. It is the same approach used by `git` (`git commit -m "..."`) and
`docker` (`docker container run ...`).

### `switch` statement

TypeScript's `switch` works exactly like Java's and C's. We switch on `subCmd` (the
sub-command string) and call the appropriate handler function. The `default` case handles
unknown sub-commands with a helpful error message.

---

## Step 5: Implement the Sub-Command Handlers

Add these four handler functions inside the exported function, after the command
registration:

```typescript
function handleAdd(name: string, ctx: any) {
    if (!name) {
        ctx.ui.notify("Usage: /bookmark add <name>", "warning");
        return;
    }
    const bookmark: Bookmark = { path: ctx.cwd, time: new Date() };
    bookmarks.set(name, bookmark);
    ctx.ui.notify(
        `Bookmark "${name}" saved:\n${bookmark.path}`,
        "success"
    );
}

function handleList(ctx: any) {
    if (bookmarks.size === 0) {
        ctx.ui.notify("No bookmarks yet. Use /bookmark add <name>.", "info");
        return;
    }
    const lines = Array.from(bookmarks.entries()).map(([name, bm]) => {
        const timeStr = bm.time.toLocaleTimeString();
        return `${name}  →  ${bm.path}  (${timeStr})`;
    });
    ctx.ui.notify(lines.join("\n"), "info");
}

function handleGo(name: string, ctx: any) {
    if (!name) {
        ctx.ui.notify("Usage: /bookmark go <name>", "warning");
        return;
    }
    const bm = bookmarks.get(name);
    if (!bm) {
        ctx.ui.notify(
            `No bookmark named "${name}". Use /bookmark list to see all.`,
            "warning"
        );
        return;
    }
    ctx.ui.notify(
        `Bookmark "${name}":\n${bm.path}\nSaved at: ${bm.time.toLocaleTimeString()}`,
        "info"
    );
}

function handleRemove(name: string, ctx: any) {
    if (!name) {
        ctx.ui.notify("Usage: /bookmark remove <name>", "warning");
        return;
    }
    const deleted = bookmarks.delete(name);
    if (deleted) {
        ctx.ui.notify(`Bookmark "${name}" removed.`, "success");
    } else {
        ctx.ui.notify(
            `No bookmark named "${name}". Use /bookmark list to see all.`,
            "warning"
        );
    }
}
```

### `ctx.cwd`

`ctx.cwd` is the current working directory of the Pi session — the directory from which
Pi was launched, or the directory the user has navigated to. This is the path we save when
creating a bookmark. It is a plain string like `"/home/alice/projects/my-app"`.

### `new Date()`

`new Date()` creates a JavaScript `Date` object representing the current date and time.
Think of it as `System.currentTimeMillis()` or `time()` plus a formatting layer.
`date.toLocaleTimeString()` formats just the time portion according to the system locale
(e.g., `"14:03:22"` or `"2:03:22 PM"`).

### `Array.from(bookmarks.entries())`

`Map.entries()` returns an iterator over `[key, value]` pairs. `Array.from()` converts
any iterable (including iterators) into a plain array. We need this because iterators do
not have `.map()` — only arrays do.

The result is an array like:
```typescript
[
    ["home", { path: "/home/alice", time: Date {...} }],
    ["project", { path: "/home/alice/projects/app", time: Date {...} }],
]
```

We then `.map()` over this to build formatted strings.

### `.join("\n")`

Joins the formatted lines array into a single string with newlines between each line.
`ctx.ui.notify()` renders newlines as actual line breaks in the notification popup, so
multi-line bookmarks lists display correctly.

### `bookmarks.delete(name)`

`Map.delete(key)` removes the entry and returns `true` if the key existed, `false` if it
did not. We use the return value to show either a success or an error message without
a separate `has()` check.

---

## Step 6: Register the Ctrl+B Shortcut

Now add the keyboard shortcut that opens an interactive picker. Add this after the command
registration:

```typescript
pi.registerShortcut("ctrl+b", {
    description: "Open bookmark picker",
    handler: async (ctx) => {
        if (bookmarks.size === 0) {
            ctx.ui.notify("No bookmarks yet. Use /bookmark add <name>.", "info");
            return;
        }

        // Build display items: "name  →  /path/to/dir  (HH:MM:SS)"
        const entries = Array.from(bookmarks.entries());
        const items = entries.map(([name, bm]) => {
            const timeStr = bm.time.toLocaleTimeString();
            return `${name}  →  ${bm.path}  (${timeStr})`;
        });

        const selected = await ctx.ui.select("Bookmarks", items);

        if (selected === undefined) {
            // User pressed Escape — no selection made
            return;
        }

        // Parse the name back out of the selected string
        const selectedName = selected.split("  →  ")[0];
        const bm = bookmarks.get(selectedName);

        if (bm) {
            ctx.ui.notify(
                `Selected: "${selectedName}"\n${bm.path}`,
                "info"
            );
        }
    },
});
```

### `ctx.ui.select(title, items)`

`ctx.ui.select()` opens a full-screen overlay dialog with a scrollable list. The user
navigates with arrow keys and confirms with Enter. The function is `async` and returns a
`Promise` that resolves to the selected string, or `undefined` if the user pressed Escape.

```typescript
const selected = await ctx.ui.select("Choose something", ["Option A", "Option B"]);
if (selected === undefined) {
    // User cancelled
} else {
    console.log("Selected:", selected); // "Option A" or "Option B"
}
```

The items array contains strings. Whatever string the user selects is returned verbatim.
If you embed additional data in the item strings (like the bookmark name at the start),
you need to parse it back out yourself — as we do with `.split("  →  ")[0]`.

### Why `await`?

`ctx.ui.select()` does not return immediately — it waits for the user to make a selection.
In traditional synchronous code, this would block the entire program. JavaScript and
TypeScript handle this with Promises and `async/await`.

When you write `const selected = await ctx.ui.select(...)`, execution pauses at that line
until the user makes a selection (or presses Escape). Other parts of Pi continue running
while your handler waits. When the user acts, your handler resumes from the line after
`await`.

This is analogous to a blocking `read()` call in C or a `Future.get()` in Java, but
implemented without blocking the thread — the runtime suspends your function and resumes
it later.

### Choosing a safe key combination

Pi uses several built-in keyboard shortcuts. Check `RESERVED_KEYS.md` before choosing a
shortcut. The format for shortcut strings is `"ctrl+letter"`, `"alt+letter"`,
`"shift+letter"`, or combinations. `"ctrl+b"` is free and easy to remember for
"bookmarks".

---

## Step 7: The Full Final Extension

Here is the complete extension (~90 lines):

```typescript
/**
 * Bookmarks — Save, list, and navigate to directories
 *
 * Commands:
 *   /bookmark add <name>    — save current working directory
 *   /bookmark list          — show all saved bookmarks
 *   /bookmark go <name>     — display a bookmark's path
 *   /bookmark remove <name> — delete a bookmark
 *
 * Shortcuts:
 *   Ctrl+B — open interactive bookmark picker
 *
 * Demonstrates: registerCommand, registerShortcut, ctx.ui.notify,
 * ctx.ui.select, ctx.cwd, Map state management.
 *
 * Usage: pi -e extensions/bookmarks.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";

interface Bookmark {
    path: string;
    time: Date;
}

export default function (pi: ExtensionAPI) {
    const bookmarks = new Map<string, Bookmark>();

    // ── Sub-command handlers ───────────────────────────────────────────────

    function handleAdd(name: string, ctx: any) {
        if (!name) {
            ctx.ui.notify("Usage: /bookmark add <name>", "warning");
            return;
        }
        const bookmark: Bookmark = { path: ctx.cwd, time: new Date() };
        bookmarks.set(name, bookmark);
        ctx.ui.notify(`Bookmark "${name}" saved:\n${bookmark.path}`, "success");
    }

    function handleList(ctx: any) {
        if (bookmarks.size === 0) {
            ctx.ui.notify("No bookmarks yet. Use /bookmark add <name>.", "info");
            return;
        }
        const lines = Array.from(bookmarks.entries()).map(([name, bm]) => {
            return `${name}  →  ${bm.path}  (${bm.time.toLocaleTimeString()})`;
        });
        ctx.ui.notify(lines.join("\n"), "info");
    }

    function handleGo(name: string, ctx: any) {
        if (!name) {
            ctx.ui.notify("Usage: /bookmark go <name>", "warning");
            return;
        }
        const bm = bookmarks.get(name);
        if (!bm) {
            ctx.ui.notify(
                `No bookmark named "${name}". Use /bookmark list to see all.`,
                "warning"
            );
            return;
        }
        ctx.ui.notify(
            `Bookmark "${name}":\n${bm.path}\nSaved at: ${bm.time.toLocaleTimeString()}`,
            "info"
        );
    }

    function handleRemove(name: string, ctx: any) {
        if (!name) {
            ctx.ui.notify("Usage: /bookmark remove <name>", "warning");
            return;
        }
        const deleted = bookmarks.delete(name);
        if (deleted) {
            ctx.ui.notify(`Bookmark "${name}" removed.`, "success");
        } else {
            ctx.ui.notify(
                `No bookmark named "${name}". Use /bookmark list to see all.`,
                "warning"
            );
        }
    }

    // ── Command registration ───────────────────────────────────────────────

    pi.registerCommand("bookmark", {
        description: "Manage bookmarks: /bookmark add <name> | list | go <name> | remove <name>",
        handler: async (args, ctx) => {
            const trimmed = (args ?? "").trim();
            const spaceIdx = trimmed.indexOf(" ");
            const subCmd = spaceIdx === -1 ? trimmed : trimmed.slice(0, spaceIdx);
            const rest   = spaceIdx === -1 ? ""      : trimmed.slice(spaceIdx + 1).trim();

            switch (subCmd) {
                case "add":    return handleAdd(rest, ctx);
                case "list":   return handleList(ctx);
                case "go":     return handleGo(rest, ctx);
                case "remove": return handleRemove(rest, ctx);
                default:
                    ctx.ui.notify(
                        `Unknown sub-command: "${subCmd}". ` +
                        `Use: add <name> | list | go <name> | remove <name>`,
                        "warning"
                    );
            }
        },
    });

    // ── Shortcut: Ctrl+B opens interactive picker ──────────────────────────

    pi.registerShortcut("ctrl+b", {
        description: "Open bookmark picker",
        handler: async (ctx) => {
            if (bookmarks.size === 0) {
                ctx.ui.notify("No bookmarks yet. Use /bookmark add <name>.", "info");
                return;
            }

            const entries = Array.from(bookmarks.entries());
            const items = entries.map(([name, bm]) => {
                return `${name}  →  ${bm.path}  (${bm.time.toLocaleTimeString()})`;
            });

            const selected = await ctx.ui.select("Bookmarks", items);
            if (selected === undefined) return;

            const selectedName = selected.split("  →  ")[0];
            const bm = bookmarks.get(selectedName);
            if (bm) {
                ctx.ui.notify(`Selected: "${selectedName}"\n${bm.path}`, "info");
            }
        },
    });

    // ── Session lifecycle ──────────────────────────────────────────────────

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        bookmarks.clear();
        ctx.ui.notify("Bookmarks ready. /bookmark add <name> | Ctrl+B to pick.", "info");
    });
}
```

---

## Step 8: Run and Test

```bash
pi -e extensions/bookmarks.ts
```

Follow this test sequence:

1. Type `/bookmark add home` — saves the current directory. You should see a green success
   notification.

2. Navigate somewhere meaningful (ask Pi to do something so you know where it is working):
   ```
   What directory am I in?
   ```

3. Type `/bookmark add project` — saves another bookmark.

4. Type `/bookmark list` — see both bookmarks in a notification with paths and times.

5. Press `Ctrl+B` — the interactive picker opens. Use arrow keys to move, Enter to select.
   After selecting, a notification shows the bookmark details.

6. Type `/bookmark remove home` — removes the first bookmark.

7. Type `/bookmark list` — only one bookmark remains.

8. Type `/bookmark go doesnotexist` — see the "not found" warning notification.

---

## Step 9: Add to themeMap.ts and justfile

In `extensions/themeMap.ts`, add:

```typescript
"bookmarks": "nord",   // clean, organized
```

In `justfile`, add:

```
# 19. Bookmarks: /bookmark add|list|go|remove + Ctrl+B picker
ext-bookmarks:
    pi -e extensions/bookmarks.ts
```

---

## Going Deeper: The Registration Order

You may have noticed that `pi.registerCommand()` and `pi.registerShortcut()` are called
before `pi.on("session_start")` in the final code. This is intentional and important.

`pi.registerCommand()` and `pi.registerShortcut()` register immediately — they do not
wait for any event. Pi's exported function runs as soon as the extension loads, so all
registrations happen before any session starts. This means commands and shortcuts are
available the moment Pi's UI appears.

`pi.on("session_start")` registers a listener that fires later. We use it only for
per-session setup (applying the theme, resetting state, showing the welcome notification).

The handlers declared as `function handleAdd(...)` etc. are hoisted by JavaScript — they
are available throughout the entire function scope, even before the lines that declare
them. This is why we can reference `handleAdd` in the switch statement even though
`handleAdd` is defined below the switch.

---

## Understanding `ctx.ui.notify()` vs `ctx.ui.select()`

| Feature | `notify()` | `select()` |
|---|---|---|
| Blocks execution? | No — fires and forgets | Yes — `await` until user acts |
| User interaction required? | No — auto-dismisses | Yes — Enter or Escape |
| Returns a value? | No | Yes — selected string or `undefined` |
| Good for | Feedback, status updates | Choosing from a list |
| Multi-line? | Yes — `\n` in message | Items are single-line strings |

Use `notify` for messages the user sees and acknowledges passively. Use `select` when you
need the user to choose something before your handler can continue.

---

## Exercises

### Exercise 1: Add /bookmark remove with list-based selection

If the user types `/bookmark remove` with no name argument, instead of showing an error,
open an interactive `ctx.ui.select()` picker listing all bookmark names. Let the user
choose which one to remove.

Hint: Check `if (!name)` — if name is empty, call `ctx.ui.select()` with the bookmark
names as items. If the user selects one, call `bookmarks.delete(selected)`.

### Exercise 2: Persist bookmarks using pi.appendEntry()

`pi.appendEntry(tag, data)` writes a structured record to the session's JSONL file. This
allows you to reconstruct state after a reload.

To persist a bookmark when adding:
```typescript
pi.appendEntry("bookmark-add", { name, path: ctx.cwd, time: new Date().toISOString() });
```

To reconstruct bookmarks on `session_start`, iterate `ctx.sessionManager.getBranch()`:
```typescript
for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type !== "appended") continue;
    if (entry.tag === "bookmark-add") {
        const data = entry.data as { name: string; path: string; time: string };
        bookmarks.set(data.name, { path: data.path, time: new Date(data.time) });
    }
    if (entry.tag === "bookmark-remove") {
        const data = entry.data as { name: string };
        bookmarks.delete(data.name);
    }
}
```

This approach uses the session log as an event log — each add and remove is an event, and
replaying them rebuilds the current state. It is the same pattern used by
`extensions/tilldone.ts` to reconstruct task lists after a session fork or switch.

### Exercise 3: Show a bookmark count in the status bar

Use `ctx.ui.setStatus()` to show the bookmark count in Pi's status line. The status line
runs across the very bottom of the screen, below the footer.

```typescript
ctx.ui.setStatus(`${bookmarks.size} bookmark${bookmarks.size !== 1 ? "s" : ""}`, "bookmarks");
```

Call this after every add, remove, and at `session_start`. The second argument is a key
for the status slot — each extension can have its own named slot in the status bar.

---

## Summary: What You Have Learned Across All Three Tutorials

Over these three tutorials you have built three complete, working Pi extensions from
scratch. Here is what each one taught:

| Tutorial | Extension | Core concepts |
|---|---|---|
| 1 | `session-info` | `setFooter`, `session_start`, timer with `setInterval`, `theme.fg()`, `visibleWidth`, `truncateToWidth` |
| 2 | `live-stats` | `setWidget`, `tool_execution_end`, ANSI background colors, `Text` component, placement options |
| 3 | `bookmarks` | `registerCommand`, `registerShortcut`, `ctx.ui.notify`, `ctx.ui.select`, `ctx.cwd`, `Map` state |

These building blocks compose into the more advanced extensions in this repository:

- `tool-counter.ts` — combines `setFooter` + `tool_execution_end` + session history
  traversal for token cost tracking
- `tilldone.ts` — combines custom AI tools + `setWidget` + `setFooter` + `tool_call`
  interception for a full task-management workflow
- `damage-control.ts` — uses `tool_call` event interception with the `isToolCallEventType`
  type guard to block dangerous operations
- `subagent-widget.ts` — spawns child processes and manages multiple widgets dynamically

The full Pi Extension API reference is at:
https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/docs/extensions.md
