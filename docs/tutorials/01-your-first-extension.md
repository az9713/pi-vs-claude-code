# Tutorial 1: Your First Extension — A Custom Footer

**Series:** Pi Extension Tutorials
**Difficulty:** Beginner
**Time:** 30–45 minutes

---

## What You Will Build

By the end of this tutorial you will have a working Pi extension called `session-info` that
replaces the default footer with a three-piece information bar:

```
claude-sonnet-4-5   Session: 04:27   Messages: 6
```

- The current model name (left)
- A live session timer that updates every second (center)
- A count of how many user messages have been sent (right)

This is a real, runnable extension you will build step by step from a blank file.

---

## What You Will Learn

- The extension entry point pattern (`export default function`)
- How Pi loads extensions via jiti (no build step required)
- The `session_start` event and the `ExtensionContext` it provides
- The `ctx.ui.setFooter()` API and what its render function receives
- Theme color tokens: `dim`, `accent`, `success`, `warning`
- How `visibleWidth()` and `truncateToWidth()` handle ANSI codes safely
- How `pi.on("input")` lets you react to every user message
- How to run your extension with `pi -e`
- How to register it in `themeMap.ts` and the `justfile`

---

## Prerequisites

- Pi, Bun, and just installed and verified (see `docs/QUICKSTART.md`)
- At least one API key configured in `.env`
- You have run `just pi` at least once and seen Pi start up

You do not need prior TypeScript experience. Every new concept is explained when it first
appears.

---

## TypeScript Crash Course for C/Java Developers

Before touching the code, here are the five TypeScript features this tutorial uses. If you
already know TypeScript, skip ahead.

### Functions are values

In C and Java, functions live at a fixed address and you pass function pointers or
interfaces around. In TypeScript, a function is just a value you can assign to a variable
or pass as an argument:

```typescript
// This is a function stored in a variable
const greet = (name: string) => `Hello, ${name}`;

// You can pass it to another function
function runGreeter(fn: (s: string) => string) {
    console.log(fn("world"));
}
runGreeter(greet); // prints: Hello, world
```

Pi uses this pattern everywhere. You call `pi.on("session_start", async (event, ctx) => {
...})` — you are passing an anonymous function as the second argument.

### `export default function`

This is how you make a value available to the outside world. Pi's loader (jiti) reads your
`.ts` file and calls whatever you `export default`. If you export a function, Pi calls it
with the `ExtensionAPI` object as the first argument:

```typescript
export default function (pi: ExtensionAPI) {
    // Pi calls this function once when the extension loads
}
```

This is equivalent to Java's `public static void main(String[] args)` — it is the entry
point.

### `import type`

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
```

The `type` keyword means "import only the type information, not any runtime code". Think
of it like importing only a C header for type-checking purposes without linking against
the library — Pi already has the library loaded at runtime.

### `async` / `await`

Most Pi event handlers are declared `async`. This is TypeScript's syntax for asynchronous
code that can pause and wait for a result without blocking the thread. The `await` keyword
is like calling `.get()` on a Java `Future`. You will see it when calling things like
`ctx.ui.input()` which needs to wait for user input.

### Template literals

Instead of string concatenation with `+`, TypeScript has template literals using backtick
quotes:

```typescript
const model = "claude-sonnet-4-5";
const pct = 30;

// Traditional concatenation:
const s1 = "Model: " + model + " (" + pct + "%)";

// Template literal (backtick):
const s2 = `Model: ${model} (${pct}%)`;
```

Both produce the same string. Template literals are easier to read and are used throughout
this project.

---

## Step 1: Create the File

Open a terminal in the project root directory. Create a new extension file:

```bash
touch extensions/session-info.ts
```

Open the file in your editor. It should be empty.

**Why this location?** Pi's extension system has no special discovery rules — you point Pi
at any `.ts` file with `-e`. By convention, this project keeps all extensions in the
`extensions/` directory alongside the existing ones. The `themeMap.ts` helper (which we
will use later) also lives there and expects to be imported with a relative path.

---

## Step 2: Write the Skeleton

Add this to `extensions/session-info.ts`:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";

export default function (pi: ExtensionAPI) {
    // We will fill this in step by step
}
```

Save the file. Now let's explain every single line.

### Line 1: `import type { ExtensionAPI } from "@mariozechner/pi-coding-agent"`

This pulls in the `ExtensionAPI` type from the Pi package. `ExtensionAPI` is the object
Pi passes into your exported function — it has `.on()` for subscribing to events,
`.registerCommand()` for slash commands, `.registerShortcut()` for keybindings, and so on.

The `type` keyword means only the type information is imported, not runtime code. This is
fine because Pi itself has the actual code loaded.

The package name `@mariozechner/pi-coding-agent` is the npm package for Pi. It was
installed when you ran `bun install`. Pi's jiti loader makes it available to your
extension automatically.

### Line 2: `import { applyExtensionDefaults } from "./themeMap.ts"`

This imports a helper function from `themeMap.ts`, which lives in the same `extensions/`
directory. The `./` prefix means "relative to this file". This function handles two
things when called: it applies the mapped theme for this extension (we will add
`session-info` to the theme map later), and it sets the terminal title bar to
`π - session-info`.

### Line 3: `import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui"`

`pi-tui` is Pi's terminal UI library. We need two utility functions:

- `visibleWidth(s: string): number` — returns the number of terminal columns a string
  occupies, **ignoring** any ANSI escape codes (the invisible bytes that produce colors).
  Plain character count alone gives the wrong answer when your string contains color codes.
- `truncateToWidth(s: string, width: number, ellipsis?: string): string` — cuts a string
  down to fit within a given terminal column count, again accounting for ANSI codes
  correctly.

We need these because footer lines must fit exactly within the terminal width, and Pi
passes the current width into your render function.

### Lines 5–7: The exported function

```typescript
export default function (pi: ExtensionAPI) {
    // ...
}
```

This is the extension entry point. Pi's loader calls this function once when it loads
your `.ts` file. The `pi` parameter is an `ExtensionAPI` instance — your handle for
everything Pi exposes to extensions. You call `pi.on(...)` to subscribe to events,
`pi.registerCommand(...)` to add slash commands, and so on.

The function does not return anything meaningful. All the work happens inside event
handlers that you register by calling `pi.on(...)`.

---

## Step 3: Add the session_start Handler

Replace the comment inside the function with this:

```typescript
export default function (pi: ExtensionAPI) {
    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
    });
}
```

Save the file. Let's break down the new lines.

### `pi.on("session_start", async (_event, ctx) => { ... })`

`pi.on()` registers a listener for a named event. `"session_start"` fires once each time
a Pi session begins — when you run `pi -e extensions/session-info.ts`, this event fires
almost immediately after startup.

The second argument is your handler function. Pi calls it with two arguments:

1. `_event` — an object with event-specific data. For `session_start` there is not much
   useful data in the event itself, which is why it is prefixed with `_` (a TypeScript
   convention meaning "I know this parameter exists but I am not using it").
2. `ctx` — an `ExtensionContext` object. This is the rich handle that gives you access to
   the current model, the session history, the UI system, the current working directory,
   and much more. Everything you do to the Pi interface goes through `ctx`.

The handler is `async` because some operations inside it — like opening dialogs — are
asynchronous and need `await`.

### `applyExtensionDefaults(import.meta.url, ctx)`

This calls the theme+title helper from `themeMap.ts`. You must pass `import.meta.url`,
which is a special value the JavaScript runtime provides: the URL of the currently
executing file. The helper uses this to derive the extension name (`"session-info"`) and
look up the correct theme in `THEME_MAP`. It also sets the terminal title to
`π - session-info`.

We pass `ctx` because the function needs it to call `ctx.ui.setTheme()` and
`ctx.ui.setTitle()`.

---

## Step 4: Add a Simple Footer

Now add the footer. Update the `session_start` handler:

```typescript
pi.on("session_start", async (_event, ctx) => {
    applyExtensionDefaults(import.meta.url, ctx);

    ctx.ui.setFooter((_tui, theme, _footerData) => ({
        dispose: () => {},
        invalidate() {},
        render(width: number): string[] {
            const model = ctx.model?.id || "no-model";
            const left = theme.fg("dim", ` ${model}`);
            return [left];
        },
    }));
});
```

### `ctx.ui.setFooter(factory)`

`setFooter` accepts a **factory function** — a function that Pi calls once to create the
footer renderer. The factory receives three arguments:

- `_tui` — a handle to request screen repaints. We will use this in the timer step.
- `theme` — the current theme object, with methods for colorizing text.
- `_footerData` — provides access to supplementary data like git branch. Not needed yet.

The factory must return an object with three methods:

- `render(width: number): string[]` — called on every terminal repaint. Receives the
  current terminal width in columns. Returns an array of strings, one per footer line.
  **Pi calls this function frequently** — on every keypress, every agent message, every
  tool call. Keep it fast.
- `invalidate()` — called when Pi wants you to discard any cached render state. Your
  render function can cache its output for performance; `invalidate()` is the signal to
  clear that cache.
- `dispose()` — called when the footer is replaced or the session ends. Use this to clean
  up timers or subscriptions. The `() => {}` is an empty function (a no-op) for now.

### `ctx.model?.id || "no-model"`

`ctx.model` is the current AI model object. The `?.` is TypeScript's optional chaining
operator — equivalent to writing `ctx.model !== null && ctx.model !== undefined ?
ctx.model.id : undefined`. The `|| "no-model"` provides a fallback string if the model
is not yet set. This pattern avoids a crash if the model object is temporarily null.

### `theme.fg("dim", ` ${model}`)`

`theme.fg(token, text)` wraps `text` in ANSI escape codes that color it using the named
theme color token. The result is a string with invisible color codes prepended and a reset
code appended. The terminal interprets these codes and displays the text in the
appropriate color.

The theme tokens used in this project are:

| Token | Visual role | Typical color |
|---|---|---|
| `dim` | Background filler, labels | Dark gray |
| `accent` | Secondary values, identifiers | Cyan |
| `success` | Primary live values, counts | Green |
| `warning` | Structural punctuation, cost | Yellow |
| `muted` | Subdued informational text | Medium gray |

The exact colors come from the active theme (e.g., synthwave, dracula, gruvbox). By using
tokens instead of hard-coded colors, your extension adapts to any theme automatically.

### `return [left]`

The render function returns an array of strings. Each string is one line of the footer.
Returning one string means a one-line footer. To show two lines, return two strings.

---

## Step 5: Run Your Extension

Run the extension:

```bash
pi -e extensions/session-info.ts
```

If you have `just` set up with your `.env`, use:

```bash
just pi  # if you want no extension
```

or run directly:

```bash
source .env && pi -e extensions/session-info.ts
```

You should see Pi start up normally, but the footer at the bottom of the screen now shows
the model name in a dim gray color instead of Pi's default footer.

**If it does not start:** Check that the file has no syntax errors. Any TypeScript error
causes Pi to print the error and abort. Read the error message — it will show a line
number. Fix it and try again.

Press `Ctrl+D` to exit Pi.

---

## Step 6: Enhance — Add a Session Timer

Now add a live timer. We need two things: a `Date.now()` call when the session starts to
record the start time, and a `setInterval` that requests a repaint every second.

Update the extension:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";

export default function (pi: ExtensionAPI) {
    let startTime = Date.now();

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        startTime = Date.now();

        ctx.ui.setFooter((tui, theme, _footerData) => {
            const timer = setInterval(() => tui.requestRender(), 1000);

            return {
                dispose: () => clearInterval(timer),
                invalidate() {},
                render(width: number): string[] {
                    const model = ctx.model?.id || "no-model";

                    const elapsed = Math.floor((Date.now() - startTime) / 1000);
                    const mins = Math.floor(elapsed / 60);
                    const secs = elapsed % 60;
                    const timerStr = `${mins}:${String(secs).padStart(2, "0")}`;

                    const left = theme.fg("dim", ` ${model}`);
                    const center = theme.fg("accent", `Session: ${timerStr}`);

                    const pad = " ".repeat(
                        Math.max(1, width - visibleWidth(left) - visibleWidth(center))
                    );
                    return [truncateToWidth(left + pad + center, width)];
                },
            };
        });
    });
}
```

### What changed

`let startTime = Date.now()` is declared **outside** the event handler. This is important:
the outer function runs once when the extension loads, and variables declared here persist
across the entire session. When `session_start` fires, we reset `startTime` to the
current timestamp.

Inside the factory function (the argument to `setFooter`), we call `setInterval`. This
schedules a callback to run every 1000 milliseconds (1 second). Each time it runs, it
calls `tui.requestRender()`, which tells Pi to call your `render` function again and
refresh the screen. Without this, the timer would only update when Pi happens to repaint
for another reason (keypress, agent message, etc.).

The `dispose` function is no longer empty — it calls `clearInterval(timer)`. This stops
the interval when the footer is replaced or the session ends, preventing a memory leak.
Always clean up timers in `dispose`.

The elapsed time calculation:
- `Date.now()` returns milliseconds since the Unix epoch (January 1, 1970). Subtracting
  `startTime` gives elapsed milliseconds.
- Dividing by 1000 and taking `Math.floor` gives whole seconds.
- `Math.floor(elapsed / 60)` gives minutes, `elapsed % 60` gives remaining seconds.
- `String(secs).padStart(2, "0")` left-pads with a zero for single-digit seconds,
  producing `"01"`, `"02"`, ..., `"59"`. This is equivalent to `printf("%02d", secs)`
  in C.

`visibleWidth(left)` and `visibleWidth(center)` measure the visible column widths of the
two strings, ignoring ANSI color codes. This lets us calculate the padding needed to push
the center string to the right side of the footer.

The `pad` calculation: `width - visibleWidth(left) - visibleWidth(center)` gives the
number of blank spaces needed between `left` and `center` so they together fill exactly
`width` columns. `Math.max(1, ...)` ensures at least one space even if the terminal is
very narrow.

`truncateToWidth(left + pad + center, width)` ensures the combined string never exceeds
the terminal width, even if the model name is very long. The concatenation uses `+` to
join the three parts into one string.

Run it:

```bash
pi -e extensions/session-info.ts
```

You should now see the timer counting up in the footer. Chat a bit — the timer keeps
running.

---

## Step 7: Enhance — Add a Message Counter

Add a user message counter. The `"input"` event fires every time the user submits a
message.

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";

export default function (pi: ExtensionAPI) {
    let startTime = Date.now();
    let messageCount = 0;

    pi.on("input", async () => {
        messageCount++;
        return { action: "continue" as const };
    });

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        startTime = Date.now();
        messageCount = 0;

        ctx.ui.setFooter((tui, theme, _footerData) => {
            const timer = setInterval(() => tui.requestRender(), 1000);

            return {
                dispose: () => clearInterval(timer),
                invalidate() {},
                render(width: number): string[] {
                    const model = ctx.model?.id || "no-model";

                    const elapsed = Math.floor((Date.now() - startTime) / 1000);
                    const mins = Math.floor(elapsed / 60);
                    const secs = elapsed % 60;
                    const timerStr = `${mins}:${String(secs).padStart(2, "0")}`;

                    const left = theme.fg("dim", ` ${model}`);
                    const center = theme.fg("accent", `Session: ${timerStr}`);
                    const right = theme.fg("success", `Messages: ${messageCount} `);

                    const pad = " ".repeat(
                        Math.max(1, width - visibleWidth(left) - visibleWidth(center) - visibleWidth(right))
                    );
                    return [truncateToWidth(left + pad + center + pad + right, width)];
                },
            };
        });
    });
}
```

Wait — there is a problem with the padding calculation above. When you have three sections
(left, center, right), you cannot naively put the same `pad` between each because the
total padding needed is `width - left - center - right`, and splitting it in two equal
pieces rarely places `center` exactly in the middle. For this tutorial we keep it simple:
the first `pad` chunk fills the gap between left and center, and then we add another pad
between center and right. This is a straightforward approach that looks clean even if
center is not mathematically centered.

A cleaner approach (used in `tool-counter.ts`) is to align left on the left edge and right
on the right edge, with `center` either skipped or incorporated into one side. For now,
the simpler layout works fine.

### The `input` event handler

`pi.on("input", async () => { ... })` fires every time the user presses Enter to send a
message. The handler must return `{ action: "continue" }` to let the message reach the AI,
or `{ action: "handled" }` to intercept it (as the purpose-gate extension does). We always
return `"continue"` — we only want to count the message, not block it.

`messageCount++` increments the counter. Because `messageCount` is declared in the outer
closure, the render function (which also closes over it) will see the updated value on the
next repaint.

**A note on event handler ordering:** `pi.on("input")` is registered before
`pi.on("session_start")`. This is fine — Pi internally separates event registration from
event firing. Both handlers are registered when Pi first calls your exported function, and
events fire later as Pi runs. The order in which you call `pi.on()` affects only the order
handlers fire for the same event, not whether they fire at all.

---

## Step 8: The Full Final Extension

Here is the complete, polished version. This adds a small visual refinement: a separator
character between sections, and a right-aligned `Messages: N` count.

```typescript
/**
 * Session Info — Model name, session timer, and message counter in the footer
 *
 * Demonstrates: setFooter, session_start event, pi.on("input"),
 * theme color tokens, visibleWidth, truncateToWidth.
 *
 * Usage: pi -e extensions/session-info.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { applyExtensionDefaults } from "./themeMap.ts";
import { truncateToWidth, visibleWidth } from "@mariozechner/pi-tui";

export default function (pi: ExtensionAPI) {
    let startTime = Date.now();
    let messageCount = 0;

    // Count every user message submitted during the session
    pi.on("input", async () => {
        messageCount++;
        return { action: "continue" as const };
    });

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        startTime = Date.now();
        messageCount = 0;

        ctx.ui.setFooter((tui, theme, _footerData) => {
            // Tick every second so the timer updates in real time
            const timer = setInterval(() => tui.requestRender(), 1000);

            return {
                dispose: () => clearInterval(timer),
                invalidate() {},
                render(width: number): string[] {
                    const model = ctx.model?.id || "no-model";

                    // Elapsed time formatted as M:SS
                    const elapsed = Math.floor((Date.now() - startTime) / 1000);
                    const mins = Math.floor(elapsed / 60);
                    const secs = elapsed % 60;
                    const timerStr = `${mins}:${String(secs).padStart(2, "0")}`;

                    const sep = theme.fg("warning", "  |  ");

                    const left =
                        theme.fg("dim", ` ${model}`) +
                        sep +
                        theme.fg("accent", `Session: ${timerStr}`);

                    const right =
                        theme.fg("success", `Messages: ${messageCount}`) +
                        theme.fg("dim", " ");

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

Run it one more time:

```bash
pi -e extensions/session-info.ts
```

Send a few messages. Watch the counter climb. Watch the timer tick. You have built a live
footer extension.

---

## Step 9: Add It to themeMap.ts

Open `extensions/themeMap.ts`. Find the `THEME_MAP` object and add an entry for your
extension:

```typescript
export const THEME_MAP: Record<string, string> = {
    "agent-chain":         "midnight-ocean",
    "agent-team":          "dracula",
    "cross-agent":         "ocean-breeze",
    "damage-control":      "gruvbox",
    "minimal":             "synthwave",
    "pi-pi":               "rose-pine",
    "pure-focus":          "everforest",
    "purpose-gate":        "tokyo-night",
    "session-info":        "nord",          // <-- add this line
    "session-replay":      "catppuccin-mocha",
    "subagent-widget":     "cyberpunk",
    "system-select":       "catppuccin-mocha",
    "theme-cycler":        "synthwave",
    "tilldone":            "everforest",
    "tool-counter":        "synthwave",
    "tool-counter-widget": "synthwave",
};
```

The theme name is the filename (without `.json`) of one of the themes in `.pi/themes/`.
Available themes: `catppuccin-mocha`, `cyberpunk`, `dracula`, `everforest`, `gruvbox`,
`midnight-ocean`, `nord`, `ocean-breeze`, `rose-pine`, `synthwave`, `tokyo-night`.

The mapping is optional — if your extension is not in the map, `applyExtensionDefaults`
falls back to `synthwave`. But adding it here makes the intent explicit.

---

## Step 10: Add a just Recipe

Open `justfile` and add a recipe for your extension. Find a good place to insert it (after
the existing numbered recipes), keeping the same comment style:

```
# 17. Session info: model name, live timer, message counter
ext-session-info:
    pi -e extensions/session-info.ts
```

Now you can run your extension with:

```bash
just ext-session-info
```

The `just` task runner reads `.env` automatically (because of `set dotenv-load := true` at
the top of the justfile), so you do not need to `source .env` separately.

---

## How It Works — Deep Dive

This section explains the render cycle in detail. Understanding this is key to writing
footers and widgets that perform well.

### The render cycle

Pi's terminal UI repaints the screen whenever something changes: a keypress, a new agent
message arriving, a tool call completing, a timer tick. When a repaint happens, Pi calls
`render(width)` on every registered footer, widget, and UI element.

Your `render` function must be **fast and pure**. Fast because it runs on every repaint.
Pure (or near-pure) because Pi may call it multiple times with the same arguments and
expects consistent output.

The `width` parameter is the current terminal column count. If the user resizes the
terminal, `invalidate()` is called (to clear any cached render output) and then
`render(width)` is called again with the new width.

### How ANSI codes affect width

Terminal colors work by embedding invisible escape sequences in the string. For example,
`\x1b[38;2;100;200;255m` means "switch the foreground color to RGB(100, 200, 255)". The
terminal interprets this and displays the following characters in that color. The sequence
itself has zero visible width.

If you use JavaScript's `.length` to measure a colored string, you get the wrong answer:

```
"hello"           → .length = 5  (correct)
"\x1b[32mhello\x1b[0m" → .length = 14 (wrong — 9 bytes are invisible ANSI codes)
```

`visibleWidth(s)` strips ANSI codes before counting characters, giving you the true column
width. `truncateToWidth(s, width)` cuts the string so its visible portion fits within
`width` columns, leaving the ANSI codes intact so colors are not broken mid-string.

### Why the timer uses `requestRender()`

The `tui` object passed to your factory function has a `requestRender()` method. Calling
it schedules a repaint on Pi's next render cycle — it does not paint immediately. Pi
batches repaints, so calling `requestRender()` many times rapidly is safe and does not
cause flickering.

Without `tui.requestRender()`, the timer display would only update when Pi happens to
repaint for another reason (like a keypress). The 1-second interval ensures the timer
updates reliably.

### The factory vs. the renderer

There are two levels of function here:

```typescript
ctx.ui.setFooter(factory);
//                ^^^^^^^
//   Pi calls this ONCE to create the footer
//
//   factory returns: { render, invalidate, dispose }
//                           ^^^^^^
//           Pi calls render() on EVERY repaint
```

Any expensive setup — like `setInterval` — belongs in the factory, not in `render`.
The factory runs once. The renderer runs constantly.

---

## Exercises

Now that you have a working extension, try these modifications to deepen your understanding.

### Exercise 1: MM:SS timer format

Change the timer to always show two digits for minutes too, so `0:05` becomes `00:05`.

Hint: The change is one line in the timer formatting code. Apply `padStart(2, "0")` to the
minutes string as well.

### Exercise 2: Add the current working directory basename

The `ctx.cwd` property on `ExtensionContext` gives you the full current working directory
path as a string. Display just the last component of the path (the directory name, not the
full path).

Hint: Import `basename` from Node.js's path module:

```typescript
import { basename } from "node:path";
```

Then use `basename(ctx.cwd)` in your render function.

### Exercise 3: Show the context usage percentage

`ctx.getContextUsage()` returns an object with a `percent` field (or `null` if not
available yet). Add a context percentage to the footer.

Hint: Look at how `extensions/minimal.ts` uses `ctx.getContextUsage()`. The pattern is:

```typescript
const usage = ctx.getContextUsage();
const pct = (usage && usage.percent !== null) ? usage.percent : 0;
```

### Exercise 4: Iterate session history for token cost

`ctx.sessionManager.getBranch()` returns an array of session entries. Each entry of type
`"message"` with `role === "assistant"` has a `usage` object containing `input`, `output`,
and `cost.total` fields. Sum `cost.total` across all assistant messages to get the total
session cost.

This is the pattern used in `extensions/tool-counter.ts` (lines 39–46). Read that file
for a complete example.

---

## What's Next

You have learned the core pattern that every Pi footer extension uses:

1. Subscribe to `session_start`
2. Call `ctx.ui.setFooter()` with a factory function
3. Inside the factory, set up any timers or subscriptions
4. Return a `{ render, invalidate, dispose }` object
5. In `render`, use `theme.fg()` for colors, `visibleWidth()` for measurement,
   `truncateToWidth()` for safety

Continue to **Tutorial 2: Building a Live Widget** to learn about `ctx.ui.setWidget()` —
which lets you add content above the editor that updates in real time as the agent works.
Widgets are more visually prominent than footers and are ideal for showing tool call
progress, agent status, and other live information.

[Tutorial 2: Building a Live Widget](./02-custom-widget.md)
