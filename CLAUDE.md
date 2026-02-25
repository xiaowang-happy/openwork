# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Is OpenWork

OpenWork is an open-source alternative to Claude Cowork/Codex. It provides a UI for running agents, skills, and MCP, powered by the OpenCode engine. Three runtime modes exist: desktop (Tauri), CLI-hosted (`openwork-orchestrator`), and cloud-hosted (den service). The app is the UI layer; the server is the execution/API layer; OpenCode is the engine.

## Common Commands

```bash
# Install dependencies
pnpm install

# Development
pnpm dev              # Desktop app (Tauri)
pnpm dev:ui           # Web UI only (Vite dev server, port 5173)
pnpm dev:web          # Same as dev:ui

# Build
pnpm build            # Full build (node scripts/build.mjs)
pnpm build:ui         # SolidJS app only

# Type checking
pnpm typecheck        # TypeScript validation for the UI package

# Tests (all route through packages/app scripts/)
pnpm test:e2e         # Full end-to-end (sessions + fs-engine + browser)
pnpm test:health      # Health check
pnpm test:sessions    # Session tests
pnpm test:refactor    # typecheck + health + sessions combined
pnpm test:events      # Event streaming
pnpm test:todos       # Todo list
pnpm test:permissions # Permission flow
pnpm test:fs-engine   # Filesystem engine

# Run a single test directly
node packages/app/scripts/<test-name>.mjs

# Per-package commands
pnpm --filter @different-ai/openwork-ui <script>
pnpm --filter openwork-orchestrator <script>
pnpm --filter openwork-server <script>
pnpm --filter @openwork/den <script>

# Version bumping (syncs app, desktop, orchestrator, tauri.conf.json, Cargo.toml)
pnpm bump:patch
pnpm bump:minor
pnpm bump:set -- 0.1.21

# Release
pnpm release:review
pnpm release:prepare
pnpm release:ship
```

## Monorepo Structure

**Package manager:** pnpm 10.27.0 with workspace protocol.

### Packages (packages/)

| Package | Name | Purpose |
|---------|------|---------|
| `packages/app` | `@different-ai/openwork-ui` | Main SolidJS UI (shared by desktop & web) |
| `packages/desktop` | `@different-ai/openwork` | Tauri 2 desktop shell (Rust) |
| `packages/web` | - | Next.js web app |
| `packages/landing` | - | Next.js landing page |
| `packages/server` | `openwork-server` | Filesystem-backed API server (Bun runtime) |
| `packages/orchestrator` | `openwork-orchestrator` | CLI host for running OpenCode + OpenWork (Bun runtime) |
| `packages/opencode-router` | - | WhatsApp/messaging bridge |

### Services (services/)

| Service | Name | Purpose |
|---------|------|---------|
| `services/den` | `@openwork/den` | Backend auth & DB (Express + MySQL + Drizzle ORM + better-auth) |
| `services/den-worker-runtime` | - | Cloud worker runtime |
| `services/openwork-share` | - | Sharing service (Vercel Blob) |

## Tech Stack

- **Frontend:** SolidJS 1.9 + Tailwind CSS 4 + Vite 6 + CodeMirror
- **Desktop:** Tauri 2 (Rust)
- **Backend services:** Express, MySQL, Drizzle ORM, better-auth
- **Server/Orchestrator binaries:** Built with Bun
- **OpenCode integration:** `@opencode-ai/sdk` (import from `@opencode-ai/sdk/v2/client` in UI code to avoid Node-only server code)
- **Routing:** `@solidjs/router` (has a pnpm patch applied)

## Architecture Essentials

### Server-Consumption Model
The OpenWork app is a **client** of the OpenWork server API surface. UI actions must map to OpenCode server APIs — do not invent parallel behavior in the app.

### OpenCode Integration
OpenWork uses `@opencode-ai/sdk/v2`:
- `createOpencode()` — launch server + create client (host mode)
- `createOpencodeClient()` — connect to existing server (client mode)
- `client.event.subscribe()` — SSE for real-time UI (streaming responses, tool calls, permissions)
- `client.session.*` — session CRUD and prompting
- `client.permission.reply()` — permission responses (once/always/reject)

### Extensibility via OpenCode Primitives
Use OpenCode's native extensibility (skills, plugins, commands, MCP, agents) rather than inventing new abstractions. These are mostly filesystem-based under `.opencode/`.

### Hot Reload (Living System)
Config changes in `.opencode/` or `opencode.json` trigger engine reloads. Reload is workspace-scoped, session-aware (queued while sessions are active), and optionally automatic.

### Web Parity
Features that read/write `.opencode/` (skills, commands, plugins, config) must route through the OpenWork server API. Tauri filesystem calls are a fallback, not a separate capability.

## Key Development Notes

- If you change `packages/server/src`, rebuild the server binary (`pnpm --filter openwork-server build:bin`) because the orchestrator runs the compiled binary, not TS sources.
- Default server binds to loopback `127.0.0.1:4096`.
- SolidJS patterns: consult `.opencode/skills/solidjs-patterns/SKILL.md` — avoid global `busy()` deadlocks; use scoped async state.
- PRDs go in `packages/app/pr/<name>.md`.
- Governance docs: `AGENTS.md`, `ARCHITECTURE.md`, `INFRASTRUCTURE.md`, `PRINCIPLES.md`, `PRODUCT.md`, `VISION.md` — read these for the "why" behind decisions.

## Den Service (services/den)

```bash
pnpm --filter @openwork/den dev          # Dev server (tsx watch)
pnpm --filter @openwork/den db:generate  # Generate Drizzle migrations
pnpm --filter @openwork/den db:migrate   # Run migrations
```

## CI/CD

GitHub Actions workflows in `.github/workflows/`. Key workflows:
- `ci.yml` — Build web, den, orchestrator on push/PR to dev
- `build-desktop.yml` — Tauri desktop builds
- Release triggered by pushing a `v*` tag

Build requirements: Node 20, pnpm 10.27.0, Bun, Rust stable.
