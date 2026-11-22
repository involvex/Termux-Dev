# AGENTS.md — Developer Guide

Project-specific guidance for AI coding agents working in this repository.

## Overview

`termux-dev` (CLI name `devx`) is an ultra-fast, terminal-native AI coding
assistant and autonomous agent built for Android Termux, Windows, macOS, and
Linux. It is a Node.js/TypeScript CLI that talks to many LLM providers
(OpenRouter, Gemini, DeepSeek, Groq, Mistral, OpenAI, Anthropic, Alibaba, and
local Ollama/LM Studio), supports MCP servers, remote control from an Android
phone, and a PLAN/AGENT dual-mode workflow.

- Package / CLI: `termux-dev` / `devx` (v2.0.3)
- Canonical docs: [README.md](README.md) (English source; eight translations live in `README_*.md`)
- Remote wire format: [docs/remote-protocol.md](docs/remote-protocol.md)

## Repository layout

| Path | Purpose |
| :--- | :--- |
| `src/cli/` | Command-line entry point and argument parsing |
| `src/core/` | Agent loop, config, sessions, cost, i18n, hooks, memory, diagnostics |
| `src/tools/` | Agent tools (`bash`, filesystem, web, background tasks, MCP, plan, etc.) |
| `src/providers/` | OpenAI- and Anthropic-compatible API clients and tool-guard logic |
| `src/prompts/` | System prompts and prompt assembly |
| `src/mcp/` | Model Context Protocol client/server integration |
| `src/permissions/` | Approval, allowlists, and security policy |
| `src/remote/` | Phone remote-control host, MQTT/WebSocket transport |
| `bin/devx.js` | Executable shim referenced by the npm `bin` field |
| `scripts/` | Build, release, and catalog-generation helpers |
| `tests/` | Vitest suites (`*.test.ts`) |
| `dist/` | Compiled output (generated; produced by `tsc`) |

## Useful Commands

All commands are defined in `package.json` and used by CI (`.github/workflows/ci.yml`).
Run them from the repository root.

```bash
# Install dependencies (CI uses npm ci for a clean, lockfile-exact install)
npm ci

# Compile TypeScript to dist/ and copy assets
npm run build

# Run the CLI from source
npm start
node ./bin/devx.js --help

# Develop with automatic recompilation
npm run dev

# Run tests once / in watch mode
npm test
npm run test:watch
```

CI-equivalent verification (match this locally before opening a PR):

```bash
npm ci
npm run build
npm test
node ./bin/devx.js --help
node ./bin/devx.js --version
node ./scripts/check-release.js   # version consistency + npm package contents
```

Debug/manual checks:

```bash
# Health diagnostics for Node, Termux, compilers, and API keys
devx /doctor

# Exercise Termux-only code paths in tests (as CI does on ubuntu + Node 22)
PREFIX=/data/data/com.termux/files/usr TERMUX_VERSION=0.118.1 npm test
```

## Technologies

- **Runtime:** Node.js >= 20 (CI tests 20.x, 22.x, and 24.x on Ubuntu, Windows, and macOS)
- **Language:** TypeScript 5.4, `strict: true`, ES2022 target, `NodeNext` modules, ESM (`"type": "module"`)
- **Build:** `tsc` plus `node ./scripts/copy-assets.js`; output in `dist/`
- **Testing:** Vitest 4 (`tests/**/*.test.ts`, Node environment, globals enabled); tests isolate history via `DEVX_HISTORY_FILE`
- **CLI parsing:** `commander`
- **Prompts / TUI:** `@clack/prompts`, `@inquirer/prompts`, `picocolors`, `qrcode-terminal`
- **Markdown rendering:** `marked`, `marked-terminal`
- **Networking:** `ws` (WebSocket), `mqtt` (phone remote relay)
- **APIs:** OpenAI (`/v1/chat/completions`) and Anthropic (`/v1/messages`) wire formats, selected per model; MCP tool integration
- **Packaging:** npm global install (`npm install -g termux-dev`); `bin/devx.js` exposes `devx` and `termux-dev`
- **License:** MIT

## Best Practices

- **Keep builds strict.** `tsconfig.json` enables `strict`, `esModuleInterop`,
  `skipLibCheck`, and consistent file casing. New code must compile with
  `npm run build` without errors — CI builds on every OS/Node combination.
- **Stay cross-platform and Node-20 compatible.** Code runs on Termux, Windows,
  macOS, and Linux, and on Node 20+. Avoid newer-only APIs and platform-specific
  path/shell assumptions; use `path` helpers and PowerShell-safe examples where relevant.
- **Test the platform-specific paths.** Termux-only behavior (notifications,
  packages, prompt) is covered by running tests with `PREFIX` and `TERMUX_VERSION`
  set. Add equivalent coverage when touching those paths.
- **Keep tests hermetic.** Never read or write a developer's real `~/.devx`
  state in tests. Use the `DEVX_HISTORY_FILE` override pattern from `vitest.config.ts`
  for any new file-writing behavior.
- **Preserve the security model.** The agent executes commands and edits files,
  so default to conservative behavior: workspace trust, confirmation for risky
  actions outside the project, secret redaction in tool output, and respect for
  `fetch_url` private-network restrictions. Never weaken these guards casually.
- **Treat YOLO/extra as unsafe by design.** `--yolo` still confirms dangerous
  commands; `--extra` skips confirmations entirely. Do not introduce code that
  silently bypasses approvals or exposes credentials (`~/.ssh`, `~/.aws`,
  `~/.devxrc.json`, `.env`, private keys).
- **Keep i18n in sync.** The interface ships in nine languages and README in
  nine languages (`README.md` is the English source). When changing user-facing
  strings or docs, update the affected translations under `src/core/locales/`
  and the `README_*.md` files.
- **Verify themes for contrast.** Every UI theme is checked in tests for readable
  contrast on the dark background (`#0a0a0c`); maintain that guarantee for new palettes.
- **Validate releases with the provided script.** Version numbers live in
  multiple files; `node ./scripts/check-release.js` verifies consistency and
  that the npm package contains the built program.

## Guidelines

- **Follow conventional commits.** History uses `feat:`, `fix:`, `docs:`, and
  `chore:` with optional scopes (for example, `feat(remote): ...`, `fix: ...`).
  Keep subjects short and describe the user-visible effect.
- **Match local checks to CI before committing/pushing.** Run `npm run build`
  and `npm test` (and `node ./scripts/check-release.js` for releases) locally;
  do not bypass git hooks or CI parity.
- **Document behavior in the English README first**, then propagate to the eight
  translations. Reference real paths and links that exist in the repo.
- **Prefer small, focused changes.** Separate provider/tool/remote concerns into
  their existing `src/` modules rather than adding logic to the CLI entry point.
- **Explain security-sensitive changes explicitly** in the PR/commit body — note
  any new approval requirement, network access, filesystem scope, or secret handling.
- **Do not commit generated or environment-specific artifacts.** `dist/`,
  `node_modules/`, and local config/session data are not source; rely on
  `.gitignore` and `.npmignore` to exclude them.
