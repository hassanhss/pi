# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

pi is a self-extensible coding agent CLI (https://pi.dev). Monorepo with lockstep versioning across 4 packages:

- **packages/ai** (`@earendil-works/pi-ai`) - Unified multi-provider LLM API (20+ providers)
- **packages/agent** (`@earendil-works/pi-agent-core`) - Agent runtime with tool calling, session management, compaction
- **packages/coding-agent** (`@earendil-works/pi-coding-agent`) - Interactive coding agent CLI (the main product)
- **packages/tui** (`@earendil-works/pi-tui`) - Terminal UI library with differential rendering

Build order matters: `tui -> ai -> agent -> coding-agent`.

## Commands

```bash
npm run check          # Lint (Biome) + pinned deps + TS imports + shrinkwrap + TS compile. Run after code changes (not docs). Does NOT run tests.
npm run build          # Build all packages in dependency order
./test.sh              # Run all non-e2e tests from repo root (sanitizes API keys)
```

Run a specific test from the package root:
```bash
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

Never run `npm test` directly (activates e2e tests when env vars are present). Never run `npm run build` unless requested.

## Architecture

- **Provider layer** (packages/ai): Lazy-registered providers with a unified streaming API. `models.generated.ts` is auto-generated from `scripts/generate-models.ts` - never edit it directly.
- **Agent loop** (packages/agent): Event-driven tool execution with context compaction to manage token budgets. Sessions stored as JSONL with branching support.
- **Extension system** (packages/coding-agent): TypeScript extensions, skills, prompt templates, and themes. Core is minimal; features belong in extensions.
- **TUI rendering** (packages/tui): Three-strategy differential rendering (only updates changed content). Native modules for platform-specific enhancements.
- **Tool registry** (packages/coding-agent/src/core/tools): Built-in tools (bash, read, write, edit, etc.) with extensible tool registration.
- **Hooks** (packages/coding-agent/src/core/hooks): Lifecycle hooks for extending agent behavior.

## Code Style

- TypeScript 5.9 with strict mode, erasable syntax only (Node strip-only mode)
- No `enum`, `namespace`/`module`, `import =`, `export =`, parameter properties in core packages
- No inline imports (`await import()`, dynamic type imports) - top-level imports only
- Tabs (width 3), 120 char line width (Biome)
- Node.js >=22.19.0, ES2022 target

## Testing

- `packages/coding-agent/test/suite/` uses `harness.ts` + faux provider (no real API keys)
- Regression tests: `packages/coding-agent/test/suite/regressions/<issue-number>-<slug>.test.ts`
- Interactive mode testing uses tmux (see AGENTS.md for the tmux workflow)

## Git

Multiple pi sessions may run concurrently. Git rules from AGENTS.md:
- Stage explicit paths only (`git add <path1> <path2>`), never `git add -A` / `git add .`
- Only commit files you changed in the current session
- Never: `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git commit --no-verify`

## Dependencies

- All direct external deps pinned to exact versions
- Install with `npm install --ignore-scripts` (don't run lifecycle scripts unless asked)
- Refresh lockfile: `npm install --package-lock-only --ignore-scripts`
- Regenerate shrinkwrap: `node scripts/generate-coding-agent-shrinkwrap.mjs`
- Pre-commit blocks lockfile changes unless `PI_ALLOW_LOCKFILE_CHANGE=1`

## Key References

- **AGENTS.md** - Full development rules (conversational style, code quality, git workflow, release process)
- **CONTRIBUTING.md** - Contribution gate (auto-close, `lgtm`/`lgtmi` approval, quality bar)
- **.pi/skills/add-llm-provider.md** - Checklist for adding new LLM providers
