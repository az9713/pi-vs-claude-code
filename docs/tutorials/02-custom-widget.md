# Tutorial 2: Building a Live Widget

**Series:** Pi Extension Tutorials
**Difficulty:** Beginner–Intermediate
**Time:** 30–45 minutes
**Prerequisite:** [Tutorial 1: Your First Extension](./01-your-first-extension.md)

---

## What You Will Build

A `live-stats` extension that adds a persistent widget above the editor. The widget tracks
every tool call the agent makes and displays per-tool counts with colored background badges
that update in real time:

```
Tools (9): [Read 4] [Bash 3] [Grep 2]
```

- `Tools (N)` shows the total call count in accent color
- Each `[Tool N]` badge has a unique dark background color
- The widget appears as soon as the first tool is called
- New tools are added automatically — you do not need to configure them

---

## What You Will Learn

- `ctx.ui.setWidget()` — the widget API, which is similar to `setFooter` but for the area
  above the editor
- The `{ placement: "belowEditor" }` option
- The `tool_execution_end` event for tracking tool usage
- ANSI background color codes (the `\x1b[48;2;R;G;Bm` format)
- The `Text` TUI component from `@mariozechner/pi-tui`
- Why widget factories are called once but render functions are called many times

---

## Where Widgets Appear

Before writing code, understand where widgets live in the Pi UI layout:

```
┌─────────────────────────────────────────────┐
│  Response area (conversation history)       │
│  ...                                        │
├─────────────────────────────────────────────┤
│  WIDGET AREA  (belowEditor placement)       │
│  Your widget renders here                  │
├─────────────────────────────────────────────┤
│  Input editor                               │
├─────────────────────────────────────────────┤
│  Footer                                     │
└─────────────────────────────────────────────┘
```

Widgets with `placement: "belowEditor"` appear between the response area and the input
editor, right above where you type. This is the most visible location in the interface —
ideal for live progress information.

Multiple widgets can be stacked. Each has a string key (like `"live-stats"`). Calling
`setWidget` with the same key again replaces the widget. Calling it with `undefined` as
the factory removes the widget.

---

## Step 1: Create the File

```bash
touch extensions/live-stats.ts
```

---

## Step 2: Write the Skeleton

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Text } from "@mariozechner/pi-tui";
import { applyExtensionDefaults } from "./themeMap.ts";

export default function (pi: ExtensionAPI) {
    // Tool call tracking state
    const counts: Record<string, number> = {};
    let total = 0;

    pi.on("tool_execution_end", async (event) => {
        // We will fill this in
    });

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
        // We will register the widget here
    });
}
```

### New concepts in this skeleton

**`const counts: Record<string, number> = {}`**

`Record<string, number>` is a TypeScript type for a dictionary (hash map) where keys are
strings and values are numbers. In Java this would be `Map<String, Integer>`. In C++ it
would be `std::unordered_map<std::string, int>`. In TypeScript/JavaScript, a plain object
`{}` serves this purpose — you can add new keys dynamically at runtime.

`const` does not mean the object is immutable — it means the variable `counts` always
points to the same object. You can still add and modify properties on that object.

**`const counts` and `let total` are declared outside the event handlers**

This is intentional. These variables live in the closure of the outer function
(`export default function`). That function runs once when the extension loads, so these
variables persist for the entire session. Every event handler and the widget's render
function all share the same `counts` and `total` because they all close over the same
scope.

In Java terms: think of the outer function body as a constructor and the closed-over
variables as instance fields. Each extension instance has one copy of these fields.

**`pi.on("tool_execution_end", async (event) => { ... })`**

`tool_execution_end` fires after every tool call completes — after the AI's `Read`,
`Bash`, `Grep`, or any other tool finishes and returns a result. The `event` object has
these useful properties:

- `event.toolName` — the name of the tool that just ran (e.g., `"Read"`, `"Bash"`)
- `event.duration` — how long the call took in milliseconds
- `event.result` — the tool's result object (usually not needed for counting)

This is different from `tool_call`, which fires before a tool runs. We use
`tool_execution_end` so we only count successful completions, not blocked or errored calls.

---

## Step 3: Implement Tool Counting

Fill in the `tool_execution_end` handler:

```typescript
pi.on("tool_execution_end", async (event) => {
    counts[event.toolName] = (counts[event.toolName] || 0) + 1;
    total++;
});
```

### Line by line

`counts[event.toolName]` reads the current count for this tool name. If the tool has
never been called before, this is `undefined`. The expression `(counts[event.toolName] || 0)`
treats `undefined` as `0` via JavaScript's short-circuit evaluation: if the left side is
falsy (`undefined`, `null`, `0`, `""`, etc.), the `||` operator returns the right side.
So on the first call for a tool, `(undefined || 0)` returns `0`, and `0 + 1` is `1`.

The result is stored back into `counts[event.toolName]`. This creates the key if it did
not exist, or updates it if it did.

`total++` increments the session total across all tools.

---

## Step 4: Build the Color Palette

Widgets display tool badges with colored backgrounds. Each tool gets a unique dark
background color. Add this above the `export default function` line:

```typescript
// Dark background colors — one per tool, cycling if more tools than colors
const PALETTE: Array<[number, number, number]> = [
    [12,  40,  80],   // deep navy
    [50,  20,  70],   // dark purple
    [10,  55,  45],   // dark teal
    [70,  30,  10],   // dark rust
    [55,  15,  40],   // dark plum
    [15,  50,  65],   // dark ocean
    [45,  45,  15],   // dark olive
    [65,  18,  25],   // dark wine
];
```

And add a helper function right after it:

```typescript
// Wrap text in an ANSI 24-bit background color
function applyBg(rgb: [number, number, number], text: string): string {
    return `\x1b[48;2;${rgb[0]};${rgb[1]};${rgb[2]}m${text}\x1b[49m`;
}
```

### How ANSI background colors work

Terminal color codes follow the format `\x1b[` + code + `m`. The `\x1b` is the escape
character (ASCII 27, also written `ESC` or `^[`). The `[` introduces the escape sequence
parameters. The `m` ends the sequence.

For 24-bit (true-color) backgrounds, the code is `48;2;R;G;B` where R, G, B are integers
0–255. For example, `\x1b[48;2;50;20;70m` sets the background to RGB(50, 20, 70) — a
dark purple.

`\x1b[49m` resets the background to the terminal's default color. Always reset after
applying a color, otherwise the color bleeds into subsequent characters.

For foreground (text) colors, the code is `38;2;R;G;B` and the reset is `\x1b[39m`.

The palette above uses dark colors intentionally — they sit below light-colored text and
remain legible in both light and dark terminal themes.

---

## Step 5: Track Per-Tool Colors

Add a map to remember which color each tool gets, and a counter to cycle through the
palette:

```typescript
export default function (pi: ExtensionAPI) {
    const counts: Record<string, number> = {};
    const toolColors: Record<string, [number, number, number]> = {};
    let total = 0;
    let colorIndex = 0;

    pi.on("tool_execution_end", async (event) => {
        // Assign a color the first time we see a tool
        if (!(event.toolName in toolColors)) {
            toolColors[event.toolName] = PALETTE[colorIndex % PALETTE.length];
            colorIndex++;
        }
        counts[event.toolName] = (counts[event.toolName] || 0) + 1;
        total++;
    });

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
    });
}
```

`event.toolName in toolColors` is the JavaScript `in` operator — it checks whether a key
exists in an object. Think of it as calling `containsKey()` on a Java Map. We only assign
a color once per tool, so the same tool always displays with the same color.

`PALETTE[colorIndex % PALETTE.length]` cycles through the palette. `%` is the modulo
operator — when `colorIndex` exceeds the palette length, it wraps back to zero. This
ensures the extension works even if more than 8 different tools are called.

---

## Step 6: Register the Widget

Now add the widget registration in the `session_start` handler. The `Text` component from
`@mariozechner/pi-tui` is a simple one-line text renderer that handles wrapping and
invalidation for you:

```typescript
pi.on("session_start", async (_event, ctx) => {
    applyExtensionDefaults(import.meta.url, ctx);

    ctx.ui.setWidget("live-stats", (_tui, theme) => {
        const text = new Text("", 1, 1);

        return {
            render(width: number): string[] {
                const entries = Object.entries(counts);

                const badges = entries.map(([name, count]) => {
                    const rgb = toolColors[name];
                    const badge = applyBg(
                        rgb,
                        // Light gray text on dark background
                        `\x1b[38;2;220;220;220m  ${name} ${count}  \x1b[39m`
                    );
                    return badge;
                });

                text.setText(
                    theme.fg("accent", `Tools (${total}):`) +
                    (entries.length > 0 ? " " + badges.join(" ") : "")
                );

                return text.render(width);
            },
            invalidate() {
                text.invalidate();
            },
        };
    });
});
```

### The `Text` component

`new Text("", 1, 1)` creates a text rendering component. The three arguments are the
initial text content and left/right padding. `Text` handles text wrapping, ANSI code
preservation, and invalidation caching for you.

`text.setText(s)` updates the text content and marks the component as needing a redraw.

`text.render(width)` returns an array of strings (one per visible line) that fit within
`width` columns.

`text.invalidate()` clears the render cache, forcing a full re-render on the next
`render()` call.

Using a `Text` component instead of building strings manually means you get correct
word-wrapping behavior if the tool list grows very long.

### `Object.entries(counts)`

`Object.entries(obj)` returns an array of `[key, value]` pairs for all enumerable
properties of the object. This is how you iterate over a JavaScript object. In Java terms,
it is like calling `entrySet()` on a `Map`. The result is an array of two-element tuples:

```typescript
const counts = { Read: 4, Bash: 3, Grep: 2 };
Object.entries(counts);
// → [["Read", 4], ["Bash", 3], ["Grep", 2]]
```

The destructuring `([name, count])` in the `.map()` callback unpacks each pair into named
variables.

### `.join(" ")`

`Array.prototype.join(separator)` concatenates all array elements into a single string
with the separator between each element. `[a, b, c].join(" ")` produces `"a b c"`.

### The `"live-stats"` key

The string `"live-stats"` is the widget's identifier. Every widget has a unique string
key. Pi uses this key to manage the widget's lifecycle — you can replace or remove it by
calling `setWidget("live-stats", ...)` again later. Choose a key that is unlikely to
collide with keys used by other extensions running simultaneously.

### The `{ placement: "belowEditor" }` option

Wait — where is it? In this version we are not passing a placement option yet, so the
widget appears in the default position (above the response area). Let's add the preferred
placement:

```typescript
ctx.ui.setWidget("live-stats", (_tui, theme) => {
    // ... factory body as above
}, { placement: "belowEditor" });
```

The third argument to `setWidget` is an options object. Setting `placement: "belowEditor"`
moves the widget to sit between the response area and the input editor — closer to where
you type, making it easier to glance at while working.

---

## Step 7: The Full Final Extension

Here is the complete, polished extension (~55 lines):

```typescript
/**
 * Live Stats Widget — Per-tool call counts with colored background badges
 *
 * Displays a widget above the editor showing tool call counts in real time.
 * Format: Tools (N): [Read 4] [Bash 3] [Grep 2]
 *
 * Demonstrates: setWidget, tool_execution_end, ANSI background colors, Text component.
 *
 * Usage: pi -e extensions/live-stats.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Text } from "@mariozechner/pi-tui";
import { applyExtensionDefaults } from "./themeMap.ts";

// Dark background colors — one per tool, cycling if more tools than entries
const PALETTE: Array<[number, number, number]> = [
    [12,  40,  80],   // deep navy
    [50,  20,  70],   // dark purple
    [10,  55,  45],   // dark teal
    [70,  30,  10],   // dark rust
    [55,  15,  40],   // dark plum
    [15,  50,  65],   // dark ocean
    [45,  45,  15],   // dark olive
    [65,  18,  25],   // dark wine
];

function applyBg(rgb: [number, number, number], text: string): string {
    return `\x1b[48;2;${rgb[0]};${rgb[1]};${rgb[2]}m${text}\x1b[49m`;
}

export default function (pi: ExtensionAPI) {
    const counts: Record<string, number> = {};
    const toolColors: Record<string, [number, number, number]> = {};
    let total = 0;
    let colorIndex = 0;

    pi.on("tool_execution_end", async (event) => {
        // Assign a color the first time this tool appears
        if (!(event.toolName in toolColors)) {
            toolColors[event.toolName] = PALETTE[colorIndex % PALETTE.length];
            colorIndex++;
        }
        counts[event.toolName] = (counts[event.toolName] || 0) + 1;
        total++;
    });

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);

        ctx.ui.setWidget("live-stats", (_tui, theme) => {
            const text = new Text("", 1, 1);

            return {
                render(width: number): string[] {
                    const entries = Object.entries(counts);

                    const badges = entries.map(([name, count]) => {
                        const rgb = toolColors[name];
                        return applyBg(
                            rgb,
                            `\x1b[38;2;220;220;220m  ${name} ${count}  \x1b[39m`
                        );
                    });

                    text.setText(
                        theme.fg("accent", `Tools (${total}):`) +
                        (entries.length > 0 ? " " + badges.join(" ") : "")
                    );

                    return text.render(width);
                },
                invalidate() {
                    text.invalidate();
                },
            };
        }, { placement: "belowEditor" });
    });
}
```

---

## Step 8: Run and Test

```bash
pi -e extensions/live-stats.ts
```

Ask Pi something that will trigger tool use:

```
Read the minimal.ts file and explain what it does.
```

Watch the widget appear as soon as the first tool call completes. Each new tool type gets
its own colored badge. Send more messages — watch the counts climb.

**Stack it with the minimal footer** to see both at once:

```bash
pi -e extensions/live-stats.ts -e extensions/minimal.ts
```

This loads both extensions simultaneously. Each extension's `session_start` fires in turn,
and both the widget and the footer appear independently.

---

## Step 9: Add to themeMap.ts and justfile

Open `extensions/themeMap.ts` and add:

```typescript
"live-stats": "synthwave",   // techy metrics
```

Open `justfile` and add:

```
# 18. Live stats: per-tool call counts in a live widget
ext-live-stats:
    pi -e extensions/live-stats.ts -e extensions/minimal.ts
```

---

## Understanding the Widget Lifecycle

The widget lifecycle mirrors the footer lifecycle from Tutorial 1, with one important
difference: a widget can be removed at runtime by calling `setWidget(key, undefined)`.

```
Extension loads
    │
    └─► outer function runs once
           │
           └─► pi.on(...) handlers are registered
                   │
                   └─► Pi fires "session_start"
                              │
                              └─► ctx.ui.setWidget("live-stats", factory)
                                           │
                                           └─► Pi calls factory() ONCE
                                                    │
                                                    └─► factory returns { render, invalidate }
                                                                 │
                                                        Pi calls render(width)
                                                        on EVERY repaint
```

The key insight: **state that should persist across repaints must live in the closure
above the `render` function**, not inside it. In our extension, `counts`, `toolColors`,
and `total` all live in the outer closure and are read by `render`. If they were declared
inside `render`, they would reset to empty on every repaint.

The `Text` component's state (its cached rendered output) lives inside the `text` object,
which is created inside the factory. This is also correct — the `text` object persists
for the lifetime of the widget, not just one render.

### Forcing a repaint when tool data updates

Did you notice that the widget updates without us calling `tui.requestRender()`? That is
because Pi's `tool_execution_end` event is part of the agent turn cycle. Pi repaints the
screen after each event handler completes. So when our `tool_execution_end` handler
increments the counters, Pi naturally repaints afterward — and that repaint calls
`render()` on our widget with the new counts.

This contrasts with Tutorial 1's timer, where we needed to explicitly call
`tui.requestRender()` because the timer fires independently of any Pi event.

---

## Exercises

### Exercise 1: Add a "most used tool" highlight

After listing all badges, add a line below showing which tool was called most:

```
Tools (9): [Read 4] [Bash 3] [Grep 2]
Most used: Read
```

Hint: Use `Object.entries(counts).sort((a, b) => b[1] - a[1])` to sort by count
descending. The first entry is the most-used tool. The `sort` comparator takes two
elements and returns a negative number (keep order), zero (equal), or positive (swap).
`b[1] - a[1]` sorts numerically descending because a larger value in position 0 produces
a positive result and gets moved earlier.

### Exercise 2: Add elapsed time since last tool call

Track `lastToolTime = Date.now()` in `tool_execution_end`. In the render function,
compute `Date.now() - lastToolTime` to show how long ago the last tool ran. Format it as
`Xs` for seconds or `Xm Xs` for minutes.

You will need to add `tui.requestRender()` on a one-second interval (as in Tutorial 1) so
the elapsed time updates even while the agent is idle.

### Exercise 3: Make the widget toggle-able with a command

Add a boolean flag `let widgetVisible = true` in the outer closure. Register a command:

```typescript
pi.registerCommand("stats", {
    description: "Toggle the live stats widget",
    handler: async (_args, ctx) => {
        widgetVisible = !widgetVisible;
        if (widgetVisible) {
            // re-register the widget
            ctx.ui.setWidget("live-stats", factory, { placement: "belowEditor" });
        } else {
            // remove the widget
            ctx.ui.setWidget("live-stats", undefined);
        }
        ctx.ui.notify(widgetVisible ? "Stats widget shown" : "Stats widget hidden", "info");
    },
});
```

Note: You will need to extract the factory function into a named variable so you can
reference it in both the initial `setWidget` call and the command handler:

```typescript
const factory = (_tui: any, theme: any) => {
    const text = new Text("", 1, 1);
    return {
        render(width: number) { /* ... */ },
        invalidate() { text.invalidate(); },
    };
};

ctx.ui.setWidget("live-stats", factory, { placement: "belowEditor" });
```

---

## What's Next

You now know how to display live, updating information both in the footer area and in a
widget above the editor. The techniques you have learned — the factory/render split, ANSI
color codes, event-driven state updates — apply to every interactive UI element in Pi.

Continue to **Tutorial 3: Slash Commands and Keyboard Shortcuts** to learn how to add
`/commands` and `Ctrl+Key` shortcuts to your extensions. Tutorial 3 builds a bookmark
manager that lets you save, list, and jump to directories — and open a picker with a
single keypress.

[Tutorial 3: Slash Commands and Keyboard Shortcuts](./03-slash-commands.md)
