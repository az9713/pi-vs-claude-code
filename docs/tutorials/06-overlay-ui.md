# Tutorial 6: Building an Interactive Overlay

**Prerequisites:** Tutorials 04 and 05, or familiarity with `pi.registerCommand()` and
`ctx.sessionManager.getBranch()`. You should understand how session event handlers and
slash commands are registered.

**What You Will Build:** A `/help` command that opens a full-screen overlay listing every
registered slash command. The overlay supports keyboard navigation (up/down arrows),
expanding a command description with Enter, and closing with Escape.

**What You Will Learn:**
- `ctx.ui.custom()` — the overlay API
- The custom component pattern: `{ render, handleInput, invalidate }`
- Key handling with `matchesKey()` and the `Key` constants object
- `Box`, `Text`, `Container`, `DynamicBorder`, `Spacer` TUI components
- Overlay options: `overlay: true`, `overlayOptions: { width, anchor }`
- Discovering registered commands from session history

---

## Background: How Overlays Work in Pi

Pi's terminal interface is drawn by a render loop. Normally the loop draws the header,
response area, widget area, editor, footer, and status line. When you open an overlay,
Pi temporarily hands control of the entire render area to your component.

The sequence:

```
User types /help and presses Enter
        |
        v
Pi calls your command handler
        |
        v
Handler calls await ctx.ui.custom(factory, options)
        |
        v
Pi calls factory(tui, theme, kb, done) — you return your component
        |
        v
Pi takes over the screen. Only your component's render() draws anything.
Only your component's handleInput() receives key events.
        |
        v
User presses Escape → your component calls done()
        |
        v
ctx.ui.custom() resolves — the overlay closes — normal Pi UI resumes
```

The `done` callback is how you close the overlay. `ctx.ui.custom()` returns a Promise
that resolves when `done` is called, so you can `await` it and do cleanup afterward.

---

## Step 1: Create the File and Imports

Create `extensions/help-overlay.ts`:

```typescript
/**
 * Help Overlay — /help command showing all registered commands with keyboard navigation.
 *
 * Press Up/Down to navigate, Enter to expand/collapse a description, Escape to close.
 *
 * Usage: pi -e extensions/help-overlay.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { DynamicBorder } from "@mariozechner/pi-coding-agent";
import {
    Box,
    Container,
    Key,
    matchesKey,
    Spacer,
    Text,
    truncateToWidth,
} from "@mariozechner/pi-tui";
import { applyExtensionDefaults } from "./themeMap.ts";
```

### Import explanations

`DynamicBorder` from `@mariozechner/pi-coding-agent`
: A horizontal rule component whose color is determined by a callback function. The
  callback receives the border character string (a line of `─` characters) and returns it
  with ANSI color applied. Pi provides this because `@mariozechner/pi-tui` does not expose
  it.

`Box` from `@mariozechner/pi-tui`
: A layout container that adds padding and an optional background transform to its
  children. Useful for indenting content or highlighting the selected row.

`Container` from `@mariozechner/pi-tui`
: A vertical stack of child components. Each child's `render(width)` is called in order
  and the results concatenated into a single `string[]`.

`Key` from `@mariozechner/pi-tui`
: An object of named key constants: `Key.up`, `Key.down`, `Key.enter`, `Key.escape`,
  `Key.tab`, and more. Used with `matchesKey()`.

`matchesKey(data, keyNameOrConstant)` from `@mariozechner/pi-tui`
: A helper that checks whether raw terminal input bytes (the `data` string received in
  `handleInput`) match a named key. Terminal key codes are platform-dependent byte
  sequences; `matchesKey` handles the decoding. You can pass either a `Key` constant
  (e.g., `Key.up`) or a string name (e.g., `"escape"`, `"ctrl+c"`).

`Spacer` from `@mariozechner/pi-tui`
: Emits one or more blank lines. `new Spacer(1)` emits a single blank line.

`truncateToWidth(text, width)` from `@mariozechner/pi-tui`
: Truncates `text` to exactly `width` visible characters, respecting ANSI escape codes
  (which contribute zero visible width). Essential for preventing line wrapping in the
  terminal.

---

## Step 2: Define the Command Item Type

Below the imports, define the shape of the data the overlay will display:

```typescript
// ── Data types ──────────────────────────────────────────────────────────────

interface CommandItem {
    name: string;
    description: string;
}
```

---

## Step 3: Build the Overlay Component Class

The overlay component is the heart of this tutorial. Add the class definition:

```typescript
// ── HelpComponent — the overlay's render and input logic ────────────────────

class HelpComponent {
    private selectedIndex: number;
    private expandedIndex: number | null;
    private scrollOffset: number;

    constructor(
        private items: CommandItem[],
        private onClose: () => void,
    ) {
        this.selectedIndex = 0;
        this.expandedIndex = null;
        this.scrollOffset = 0;
    }
```

### Why a class?

In the TUI, the overlay needs to maintain state between render calls: which item is
selected, whether any item is expanded, and how far the list has scrolled. A class with
private fields is the natural fit. The `private` keyword in TypeScript is a compile-time
access modifier — it does not affect the JavaScript output, but it communicates intent and
catches accidental field access at edit time.

`onClose` is stored as a private field so `handleInput` can call it when the user presses
Escape. This is the callback pattern: the component does not need to know anything about
Pi's overlay system; it just calls `onClose()` when it wants to close.

### handleInput — responding to keyboard events

```typescript
    handleInput(data: string, tui: { requestRender(): void }): void {
        if (matchesKey(data, Key.up)) {
            this.selectedIndex = Math.max(0, this.selectedIndex - 1);
            this.updateScroll();
        } else if (matchesKey(data, Key.down)) {
            this.selectedIndex = Math.min(
                this.items.length - 1,
                this.selectedIndex + 1,
            );
            this.updateScroll();
        } else if (matchesKey(data, Key.enter)) {
            // Toggle expansion: if already expanded, collapse; otherwise expand
            this.expandedIndex =
                this.expandedIndex === this.selectedIndex
                    ? null
                    : this.selectedIndex;
        } else if (matchesKey(data, Key.escape) || matchesKey(data, "ctrl+c")) {
            this.onClose();
            return; // Do not request a render after closing
        }

        // Tell Pi to redraw the overlay
        tui.requestRender();
    }
```

`handleInput` receives two arguments: the raw terminal input bytes as a `string`, and the
`tui` object. The `tui` object's primary purpose in this context is `requestRender()`,
which signals Pi to call `render()` again on the next frame. Without this call, the
screen would not update after a keypress.

`Math.max(0, selectedIndex - 1)` clamps the selection at 0. `Math.min(items.length - 1,
selectedIndex + 1)` clamps it at the last item. This is the same pattern used in
`session-replay.ts`.

### updateScroll — keeping the selected item visible

```typescript
    private updateScroll(): void {
        const PAGE_SIZE = 8; // Approximate visible items per screen
        if (this.selectedIndex < this.scrollOffset) {
            this.scrollOffset = this.selectedIndex;
        } else if (this.selectedIndex >= this.scrollOffset + PAGE_SIZE) {
            this.scrollOffset = this.selectedIndex - PAGE_SIZE + 1;
        }
    }
```

This is a simple scroll window algorithm. If the selected item scrolls above the top of
the visible window, the window shifts up. If it scrolls below the bottom, the window
shifts down. The constant `PAGE_SIZE = 8` is an approximation — the actual visible count
depends on the terminal height and item heights.

### render — producing the visible lines

```typescript
    render(width: number, theme: any): string[] {
        const container = new Container();

        // ── Header ──────────────────────────────────────────────────────────
        container.addChild(
            new DynamicBorder((s) => theme.fg("accent", s)),
        );
        container.addChild(
            new Text(
                " " + theme.fg("accent", theme.bold("HELP")) +
                theme.fg("dim", "  ") +
                theme.fg("muted", `${this.items.length} commands`) +
                theme.fg("dim", "  ·  ↑↓ navigate  ·  Enter expand  ·  Esc close"),
                1,
                0,
            ),
        );
        container.addChild(
            new DynamicBorder((s) => theme.fg("borderMuted", s)),
        );
        container.addChild(new Spacer(1));

        // ── Items ────────────────────────────────────────────────────────────
        const visibleItems = this.items.slice(this.scrollOffset);

        for (let i = 0; i < visibleItems.length; i++) {
            const absoluteIndex = i + this.scrollOffset;
            const item = visibleItems[i];
            const isSelected = absoluteIndex === this.selectedIndex;
            const isExpanded = absoluteIndex === this.expandedIndex;

            // Box adds a highlight background when this item is selected.
            // The third argument to Box is a "transform" function applied to each
            // rendered line — we use it to apply a background color.
            const row = new Box(
                1, // paddingLeft
                0, // paddingRight
                isSelected
                    ? (s: string) => theme.bg("selectedBg", s)
                    : undefined,
            );

            // Build the title line for this item
            const prefix = isSelected
                ? theme.fg("accent", "> ")
                : theme.fg("dim",    "  ");
            const nameText = isSelected
                ? theme.fg("accent", theme.bold(`/${item.name}`))
                : theme.fg("muted", `/${item.name}`);
            const expandHint = isExpanded
                ? theme.fg("dim", " [↑ collapse]")
                : theme.fg("dim", " [↵ expand]");

            row.addChild(new Text(prefix + nameText + expandHint, 0, 0));

            if (isExpanded) {
                // Show the full description below the name
                row.addChild(new Spacer(1));
                row.addChild(
                    new Text(
                        theme.fg("dim", "  " + item.description),
                        0,
                        0,
                    ),
                );
                row.addChild(new Spacer(1));
            } else {
                // Show a truncated preview of the description on the same visual block
                const preview = truncateToWidth(
                    theme.fg("dim", "  " + item.description),
                    width - 6,
                );
                row.addChild(new Text(preview, 0, 0));
            }

            container.addChild(row);
        }

        // ── Footer ──────────────────────────────────────────────────────────
        container.addChild(new Spacer(1));
        container.addChild(
            new DynamicBorder((s) => theme.fg("accent", s)),
        );

        return container.render(width);
    }
```

### TUI component system explained

Pi's TUI uses a tree of components. Every component has a `render(width: number):
string[]` method that returns an array of strings — one per terminal line.

`Container` is the most fundamental layout component. Calling `container.addChild(child)`
appends a child. When `container.render(width)` is called, it calls `render(width)` on
each child in order and concatenates the returned arrays.

`DynamicBorder` renders a horizontal line of `─` characters at the given width. It
accepts a transform function (a callback) that wraps the line string in ANSI color codes.
The callback receives the raw line and must return the colored version.

`Box` wraps its children with optional left/right padding (in spaces) and an optional
line-transform function applied to each rendered line. Here we use the transform to apply
a background color to the entire row when the item is selected. The `theme.bg("selectedBg",
s)` call wraps `s` in ANSI background color escape codes for the `selectedBg` color token.

`Spacer(n)` emits `n` blank lines.

`Text(content, paddingLeft, paddingRight)` is a leaf node that holds a string. The
padding arguments add spaces before and after on each line.

### The `invalidate` method

```typescript
    invalidate(): void {
        // This component has no render cache, so invalidate is a no-op.
        // If you cache rendered lines (as TillDoneListComponent does), clear
        // the cache here so the next render() call produces fresh output.
    }
}
```

`invalidate()` is called by Pi when something external changes that would affect the
component's output — for example, a theme change. If your `render()` function caches its
output (stores the result of a previous call and returns it when the width has not
changed), you must clear that cache in `invalidate()`. See `TillDoneListComponent.render()`
in `tilldone.ts` for an example of this optimization.

---

## Step 4: The Extension Entry Point and Command Registration

```typescript
// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
    });

    pi.registerCommand("help", {
        description: "Show all registered slash commands",
        handler: async (_args, ctx) => {
            if (!ctx.hasUI) {
                ctx.ui.notify("/help requires interactive mode (no --headless)", "warning");
                return;
            }

            // ── Collect commands from session entries ──────────────────────
            //
            // Pi does not expose a direct API to list registered commands. However,
            // commands are recorded in the session as metadata entries when they are
            // invoked, and the commands we register ourselves are known at extension
            // load time.
            //
            // The simpler approach for a help overlay: declare the list manually.
            // In a real extension, you could read session entries with
            // ctx.sessionManager.getBranch() to find commands that have been used,
            // or build a registry in closure scope as commands are registered.

            const commands: CommandItem[] = [
                {
                    name: "help",
                    description: "Show this help overlay. Navigate with ↑↓, expand with Enter, close with Esc.",
                },
                // Add entries for any other commands your extension stack registers.
                // If you load safe-bash alongside this extension, add:
                // { name: "safe-bash-log", description: "Show the safe-bash audit log." },
                // If you load tilldone, add:
                // { name: "tilldone", description: "Show the TillDone task list overlay." },
            ];

            // Include a note about Pi's built-in commands
            commands.push({
                name: "replay",
                description: "(session-replay extension) Open a scrollable timeline of the session history.",
            });
            commands.push({
                name: "theme",
                description: "(theme-cycler extension) Select or switch the active color theme.",
            });

            if (commands.length === 0) {
                ctx.ui.notify("No commands registered.", "warning");
                return;
            }

            // ── Open the overlay ───────────────────────────────────────────
            //
            // ctx.ui.custom() takes a factory function and returns a Promise.
            // The factory receives:
            //   tui    — the TUI instance (use tui.requestRender() to trigger redraws)
            //   theme  — the current Theme object
            //   kb     — keyboard state (not needed here)
            //   done   — call done() to close the overlay and resolve the Promise
            //
            // The factory must return an object with render, handleInput, and invalidate.

            await ctx.ui.custom(
                (tui, theme, _kb, done) => {
                    const component = new HelpComponent(commands, () => done(undefined));
                    return {
                        render: (width: number) => component.render(width, theme),
                        handleInput: (data: string) => component.handleInput(data, tui),
                        invalidate: () => component.invalidate(),
                    };
                },
                {
                    overlay: true,
                    overlayOptions: {
                        width: "70%",  // Take up 70% of the terminal width
                        anchor: "center", // Center horizontally
                    },
                },
            );
        },
    });
}
```

### `ctx.ui.custom()` in detail

The first argument is the factory function. Pi calls it once when the overlay opens:

```
factory(tui, theme, kb, done) → { render, handleInput, invalidate }
```

- `tui` — the TUI engine. Call `tui.requestRender()` after any state change to trigger a
  redraw.
- `theme` — the current Theme object. Pass it to your component for color application.
- `kb` — the keyboard state object. Rarely needed directly; `matchesKey` handles key
  decoding.
- `done` — call `done(value)` to close the overlay. The value is the resolved value of
  the Promise. Since this overlay returns no data, we pass `undefined`.

The returned object is your component interface:
- `render(width)` — called every frame that needs a redraw. Returns `string[]`.
- `handleInput(data)` — called for every key event while the overlay is open.
- `invalidate()` — called when external state changes (e.g., theme switch).

The second argument to `ctx.ui.custom()` is an options object:
- `overlay: true` — required to show this as a floating overlay rather than replacing the
  entire screen. The main Pi UI is visible around the overlay.
- `overlayOptions.width` — can be a percentage string `"70%"` or a number of columns.
- `overlayOptions.anchor` — `"center"` centers the overlay horizontally. Other values
  depend on the Pi version; `"center"` is the most common.

### `ctx.hasUI`

Some Pi invocations run in headless mode (no terminal). The `ctx.hasUI` boolean tells you
whether interactive UI is available. Always check it before calling UI-only APIs like
`ctx.ui.custom()`, `ctx.ui.confirm()`, and `ctx.ui.input()`. If not in UI mode, fall back
to `ctx.ui.notify()` with an error message.

---

## Step 5: Run and Test

```bash
pi -e extensions/help-overlay.ts
```

Once Pi starts, type `/help` and press Enter. The overlay opens showing the command list.
Press Up and Down to move the selection cursor. Press Enter on a command to expand its
description. Press Enter again to collapse it. Press Escape to close.

Stack the extension with others to populate the command list:

```bash
pi -e extensions/help-overlay.ts -e extensions/safe-bash.ts -e extensions/tilldone.ts
```

Then update the `commands` array in the handler to include entries for the other
extensions' commands.

---

## Complete File

```typescript
/**
 * Help Overlay — /help command showing all registered commands with keyboard navigation.
 *
 * Usage: pi -e extensions/help-overlay.ts
 */

import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { DynamicBorder } from "@mariozechner/pi-coding-agent";
import {
    Box,
    Container,
    Key,
    matchesKey,
    Spacer,
    Text,
    truncateToWidth,
} from "@mariozechner/pi-tui";
import { applyExtensionDefaults } from "./themeMap.ts";

// ── Data types ──────────────────────────────────────────────────────────────

interface CommandItem {
    name: string;
    description: string;
}

// ── HelpComponent ───────────────────────────────────────────────────────────

class HelpComponent {
    private selectedIndex: number;
    private expandedIndex: number | null;
    private scrollOffset: number;

    constructor(
        private items: CommandItem[],
        private onClose: () => void,
    ) {
        this.selectedIndex = 0;
        this.expandedIndex = null;
        this.scrollOffset = 0;
    }

    handleInput(data: string, tui: { requestRender(): void }): void {
        if (matchesKey(data, Key.up)) {
            this.selectedIndex = Math.max(0, this.selectedIndex - 1);
            this.updateScroll();
        } else if (matchesKey(data, Key.down)) {
            this.selectedIndex = Math.min(
                this.items.length - 1,
                this.selectedIndex + 1,
            );
            this.updateScroll();
        } else if (matchesKey(data, Key.enter)) {
            this.expandedIndex =
                this.expandedIndex === this.selectedIndex
                    ? null
                    : this.selectedIndex;
        } else if (matchesKey(data, Key.escape) || matchesKey(data, "ctrl+c")) {
            this.onClose();
            return;
        }
        tui.requestRender();
    }

    private updateScroll(): void {
        const PAGE_SIZE = 8;
        if (this.selectedIndex < this.scrollOffset) {
            this.scrollOffset = this.selectedIndex;
        } else if (this.selectedIndex >= this.scrollOffset + PAGE_SIZE) {
            this.scrollOffset = this.selectedIndex - PAGE_SIZE + 1;
        }
    }

    render(width: number, theme: any): string[] {
        const container = new Container();

        container.addChild(
            new DynamicBorder((s) => theme.fg("accent", s)),
        );
        container.addChild(
            new Text(
                " " + theme.fg("accent", theme.bold("HELP")) +
                theme.fg("dim", "  ") +
                theme.fg("muted", `${this.items.length} commands`) +
                theme.fg("dim", "  ·  ↑↓ navigate  ·  Enter expand  ·  Esc close"),
                1,
                0,
            ),
        );
        container.addChild(
            new DynamicBorder((s) => theme.fg("borderMuted", s)),
        );
        container.addChild(new Spacer(1));

        const visibleItems = this.items.slice(this.scrollOffset);

        for (let i = 0; i < visibleItems.length; i++) {
            const absoluteIndex = i + this.scrollOffset;
            const item = visibleItems[i];
            const isSelected = absoluteIndex === this.selectedIndex;
            const isExpanded = absoluteIndex === this.expandedIndex;

            const row = new Box(
                1,
                0,
                isSelected
                    ? (s: string) => theme.bg("selectedBg", s)
                    : undefined,
            );

            const prefix = isSelected
                ? theme.fg("accent", "> ")
                : theme.fg("dim",    "  ");
            const nameText = isSelected
                ? theme.fg("accent", theme.bold(`/${item.name}`))
                : theme.fg("muted", `/${item.name}`);
            const expandHint = isExpanded
                ? theme.fg("dim", " [↑ collapse]")
                : theme.fg("dim", " [↵ expand]");

            row.addChild(new Text(prefix + nameText + expandHint, 0, 0));

            if (isExpanded) {
                row.addChild(new Spacer(1));
                row.addChild(
                    new Text(theme.fg("dim", "  " + item.description), 0, 0),
                );
                row.addChild(new Spacer(1));
            } else {
                const preview = truncateToWidth(
                    theme.fg("dim", "  " + item.description),
                    width - 6,
                );
                row.addChild(new Text(preview, 0, 0));
            }

            container.addChild(row);
        }

        container.addChild(new Spacer(1));
        container.addChild(
            new DynamicBorder((s) => theme.fg("accent", s)),
        );

        return container.render(width);
    }

    invalidate(): void {
        // No render cache — nothing to clear
    }
}

// ── Extension entry point ───────────────────────────────────────────────────

export default function (pi: ExtensionAPI) {

    pi.on("session_start", async (_event, ctx) => {
        applyExtensionDefaults(import.meta.url, ctx);
    });

    pi.registerCommand("help", {
        description: "Show all registered slash commands",
        handler: async (_args, ctx) => {
            if (!ctx.hasUI) {
                ctx.ui.notify("/help requires interactive mode", "warning");
                return;
            }

            const commands: CommandItem[] = [
                {
                    name: "help",
                    description:
                        "Show this help overlay. Navigate with ↑↓, expand with Enter, close with Esc.",
                },
                {
                    name: "replay",
                    description:
                        "(session-replay extension) Open a scrollable timeline of the session history.",
                },
                {
                    name: "theme",
                    description:
                        "(theme-cycler extension) Select or switch the active color theme.",
                },
                {
                    name: "tilldone",
                    description:
                        "(tilldone extension) Show the TillDone task list overlay.",
                },
            ];

            await ctx.ui.custom(
                (tui, theme, _kb, done) => {
                    const component = new HelpComponent(commands, () => done(undefined));
                    return {
                        render: (width: number) => component.render(width, theme),
                        handleInput: (data: string) => component.handleInput(data, tui),
                        invalidate: () => component.invalidate(),
                    };
                },
                {
                    overlay: true,
                    overlayOptions: {
                        width: "70%",
                        anchor: "center",
                    },
                },
            );
        },
    });
}
```

---

## Deep Dive: How Overlays Work

The `ctx.ui.custom()` API wraps Pi's internal TUI render loop. When called, Pi pushes
your component onto a render stack. On every terminal refresh cycle, Pi checks the stack.
If your component is on top, it calls `component.render(width)` and writes the returned
lines to the terminal, replacing whatever was there before.

Key events arrive from the terminal's input stream as raw byte sequences. Pi passes them
to `component.handleInput(data)`. After your handler runs, if you called
`tui.requestRender()`, Pi schedules a redraw for the next cycle.

When you call `done()`, Pi pops your component off the render stack. The normal Pi UI
resumes rendering. The Promise returned by `ctx.ui.custom()` resolves with the value you
passed to `done()`.

This architecture means overlays are always synchronous from the component's perspective
— `render()` and `handleInput()` are regular function calls, not async operations. Only
the outer `await ctx.ui.custom(...)` is async, because it waits for `done()` to be called.

The `invalidate()` method exists to support render caching. If your `render()` function
is expensive (e.g., it lays out text that requires measuring string widths), you can cache
its output and return the cached version on subsequent calls when the width has not
changed. When Pi knows the cache must be discarded — for example, because the theme
changed — it calls `invalidate()`. See `TillDoneListComponent.render()` in `tilldone.ts`
for a concrete example using `cachedLines` and `cachedWidth`.

---

## Exercises

**Exercise 1 — Add a search filter**

Add a `filterText: string` field to `HelpComponent`. In `handleInput`, intercept printable
character keypresses and append them to `filterText`. Handle Backspace by removing the
last character. Filter `this.items` in `render()` to only show items where
`item.name.includes(filterText) || item.description.includes(filterText)`. Display the
filter text in the header area using a `Text` component.

To detect printable characters, check: `data.length === 1 && data.charCodeAt(0) >= 32`.
To detect Backspace, use: `matchesKey(data, "backspace")`.

**Exercise 2 — Color-code by category**

Add a `category: "builtin" | "extension"` field to `CommandItem`. In `render()`, apply
different colors based on the category: `theme.fg("success", ...)` for built-in commands,
`theme.fg("accent", ...)` for extension commands. Add a legend line in the header
explaining the color coding.

**Exercise 3 — Keyboard shortcut to open the overlay**

Instead of (or in addition to) the `/help` command, register a keyboard shortcut using
`pi.registerShortcut("ctrl+h", { ... })`. In the shortcut handler, call the same
`ctx.ui.custom(...)` logic used in the command handler. Extract the overlay-opening logic
into a shared function to avoid duplication.

---

## What's Next

You have now completed all three hands-on tutorials in this series:

- **Tutorial 4** — taught you how the AI gets access to new capabilities through
  `pi.registerTool()`, TypeBox schemas, and the `details` field for state persistence.
- **Tutorial 5** — taught you how to intercept tool calls before they run, block
  dangerous operations, and log everything to the session.
- **Tutorial 6** — taught you how to build rich interactive UI that takes over the
  terminal with keyboard navigation and styled components.

These three patterns — tools, event interception, and overlay UI — cover the majority of
what any Pi extension does. The remaining extensions in this repository combine these
patterns in more complex ways:

| Extension | Patterns used |
|---|---|
| `tilldone.ts` | Tool + gate (Tutorial 4+5 combined) + overlay (Tutorial 6) + widget |
| `damage-control.ts` | Event interception (Tutorial 5) + YAML rule loading |
| `session-replay.ts` | Overlay (Tutorial 6) + session branch traversal |
| `theme-cycler.ts` | Commands + shortcuts + widget (simpler UI than an overlay) |
| `subagent-widget.ts` | Commands + spawning child Pi processes + widgets |
| `agent-team.ts` | All of the above + multi-agent orchestration |

The recommended next read is `tilldone.ts` in full. It is the most pedagogically complete
extension in the repository: it uses every concept from Tutorials 4, 5, and 6, plus
footer customization (from Tutorial 2) and widget rendering (from Tutorial 3). Now that
you have built the components from scratch, the tilldone source should read as familiar
territory.
