# Pi vs CC — Extension Playground

Pi Coding Agent extension examples and experiments.

## Tooling
- **Package manager**: `bun` (not npm/yarn/pnpm)
- **Task runner**: `just` (see justfile)
- **Extensions run via**: `pi -e extensions/<name>.ts`

## Project Structure
- `extensions/` — Pi extension source files (.ts), one file per extension
- `extensions/themeMap.ts` — Shared theme mapping, applyExtensionDefaults helper
- `specs/` — Feature specifications
- `.pi/agents/` — Agent definitions (.md with frontmatter) for agent-team and agent-chain
- `.pi/agents/pi-pi/` — Expert agents for the pi-pi meta-agent
- `.pi/agents/teams.yaml` — Team compositions for agent-team extension
- `.pi/agents/agent-chain.yaml` — Pipeline definitions for agent-chain extension
- `.pi/themes/` — 11 custom JSON themes (51 color tokens each)
- `.pi/damage-control-rules.yaml` — Safety rules for damage-control extension
- `.pi/agent-sessions/` — Ephemeral session files (gitignored)

## Conventions
- Extensions are standalone .ts files loaded by Pi's jiti runtime (no build step)
- Available imports: `@mariozechner/pi-coding-agent`, `@mariozechner/pi-tui`, `@mariozechner/pi-ai`, `@sinclair/typebox`, plus any deps in package.json
- Register tools at the TOP LEVEL of the extension function (not inside event handlers)
- Use `isToolCallEventType()` for type-safe tool_call event narrowing
- Use `applyExtensionDefaults(import.meta.url, ctx)` in session_start for theme + title setup
- Footer renderers return `string[]` (one string per line)
- Use TypeBox (`@sinclair/typebox`) for tool parameter schemas
- Theme color tokens: `success` (green), `accent` (highlight), `warning` (yellow), `dim` (gray), `muted` (lighter gray)
- Agent definitions use frontmatter format: `---\nname: ...\ndescription: ...\ntools: ...\n---\n<system prompt>`

## Documentation
- **docs/QUICKSTART.md** — Setup and 10 hands-on use cases for new users
- **docs/ARCHITECTURE.md** — Full architecture with ASCII diagrams and communication flows
- **docs/DEVELOPER_GUIDE.md** — API reference, extension anatomy, tutorials, patterns
- **docs/STUDY_PLAN.md** — Zero-to-hero learning plan (9 phases)
- **COMPARISON.md** — Claude Code vs Pi Agent feature matrix
- **THEME.md** — Color token conventions
- **TOOLS.md** — Built-in tool signatures

## Key Patterns
- **State reconstruction**: Scan session history for previous tool results (see tilldone.ts)
- **Tool blocking gate**: Return `{block: true, reason}` from `tool_call` handler (see tilldone.ts, damage-control.ts)
- **System prompt injection**: Return `{systemPrompt}` from `before_agent_start` (see agent-team.ts, purpose-gate.ts)
- **Subagent spawning**: `spawn("pi", ["--mode", "json", "-p", ...])` and parse JSONL stdout (see agent-team.ts, subagent-widget.ts)
- **Fire-and-forget**: Start async subprocess without await, deliver results via `pi.sendMessage({}, {triggerTurn: true})`
