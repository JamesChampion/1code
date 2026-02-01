# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**1Code** — An Electron desktop app providing a visual UI for AI-powered code assistance. Users create chat sessions linked to local project folders, interact with Claude in Plan or Agent mode, and see real-time tool execution (bash, file edits, web search). Built by the 21st.dev team.

## Environment Setup

**Required for development:**

```bash
# Install Bun (package manager)
curl -fsSL https://bun.sh/install | bash

# Clone and setup
git clone <repo>
cd 1code
bun install
bun run claude:download  # REQUIRED before first dev run
```

**Optional environment variables** (`.env.local`):

```bash
# Analytics (disabled by default in dev)
MAIN_VITE_POSTHOG_KEY=your_key
VITE_POSTHOG_KEY=your_key

# Error tracking (disabled by default)
MAIN_VITE_SENTRY_DSN=your_dsn

# OpenAI voice transcription
MAIN_VITE_OPENAI_API_KEY=your_key

# Custom Claude endpoint (for self-hosted)
ANTHROPIC_BASE_URL=http://localhost:8080
ANTHROPIC_API_KEY=your_key
```

## Commands

```bash
# Development
bun install                  # Install deps (rebuilds better-sqlite3, node-pty)
bun run claude:download      # Download Claude CLI binary (required first time)
bun run dev                  # Start Electron with hot reload
bun run ts:check             # TypeScript validation (uses tsgo)

# Build & Package
bun run build                # Compile TypeScript
bun run package              # Package for current platform (dir output)
bun run package:mac          # macOS (DMG + ZIP, arm64 + x64)
bun run package:win          # Windows (NSIS + portable)
bun run package:linux        # Linux (AppImage + DEB)

# Database
bun run db:generate          # Generate Drizzle migrations from schema
bun run db:push              # Push schema directly (dev only)
bun run db:studio            # Open Drizzle Studio

# Release (full pipeline: build, sign, notarize, upload to R2 CDN)
bun run release
```

There is no test suite or linter configured in this project.

## Troubleshooting

**"Agent chat doesn't work" after fresh clone:**
```bash
bun run claude:download  # Downloads required Claude binary
```

**"better-sqlite3" build errors:**
```bash
bun install  # Rebuilds native modules via postinstall
# If fails: delete node_modules and reinstall
```

**"Protocol handler not working" on macOS:**
- First launch may not register protocol
- Click "Sign in" again to trigger registration
- Restart app if needed

**TypeScript errors:**
```bash
bun run ts:check  # Validate types without building
```

**Database schema mismatch:**
- Migrations auto-run on startup
- Dev: migrations in `drizzle/`
- Prod: migrations in `resources/migrations/`
- Both must stay in sync

## Architecture

Three-process Electron app with tRPC for type-safe IPC:

- **Main process** (`src/main/`) — Electron lifecycle, SQLite database, tRPC routers, auth, Claude SDK integration, git operations, terminal management
- **Preload** (`src/preload/index.ts`) — Context-isolated IPC bridge exposing `window.desktopApi` (window controls, auth, file watchers, auto-update) and tRPC bridge
- **Renderer** (`src/renderer/`) — React 19 UI with Jotai (UI state), Zustand (complex stores), and React Query via tRPC (server state)

### Main Process Key Areas

| Directory | Purpose |
|-----------|---------|
| `src/main/lib/db/` | Drizzle ORM + better-sqlite3, WAL mode, auto-migration on startup |
| `src/main/lib/trpc/routers/` | 15 tRPC routers: projects, chats, claude, claudeCode, claudeSettings, ollama, terminal, external, files, debug, skills, agents, worktreeConfig, commands, voice |
| `src/main/lib/claude/` | Claude SDK integration (env, binary, offline fallback, raw logger, transform) |
| `src/main/lib/git/` | Git operations, worktree management, file watchers, GitHub integration |
| `src/main/lib/terminal/` | Terminal sessions via node-pty, port scanning |
| `src/main/auth-manager.ts` | OAuth token management with auto-refresh (5 min before expiry) |
| `src/main/auth-store.ts` | Encrypted credential storage via Electron safeStorage |

### Renderer Key Areas

| Directory | Purpose |
|-----------|---------|
| `src/renderer/features/agents/` | Core chat interface: message list, input area, tool renderers (40+ tool components), slash commands, mentions |
| `src/renderer/features/agents/atoms/` | Jotai atoms: `selectedAgentChatIdAtom` (window-scoped), preview settings per chat |
| `src/renderer/features/agents/stores/` | Zustand stores: sub-chat tabs, message caching, streaming status |
| `src/renderer/features/sidebar/` | Chat list, archive, navigation |
| `src/renderer/features/changes/` | Git UI: staging, commits, diff views |
| `src/renderer/features/terminal/` | xterm.js wrapper with search and link detection |
| `src/renderer/features/details-sidebar/` | Collapsible right sidebar (changes, diff, terminal, plan widgets) |
| `src/renderer/components/dialogs/` | Settings dialog (profile, appearance, MCP, skills, agents, models, debug) |

### Database

SQLite at `{userData}/data/agents.db`. Schema source of truth: `src/main/lib/db/schema/index.ts`

```
projects    → id, name, path (unique), git remote fields, timestamps
chats       → id, name, projectId (FK), worktree fields, PR tracking, timestamps
sub_chats   → id, name, chatId (FK), sessionId, streamId, mode, messages (JSON), timestamps
claudeCodeCredentials → single-row OAuth token storage
```

Migrations live in `drizzle/` (dev) or `resources/migrations` (packaged). Auto-run on startup via `initDatabase()`.

## Common Development Tasks

**Add a new tRPC router:**
1. Create router: `src/main/lib/trpc/routers/my-feature.ts`
2. Export from: `src/main/lib/trpc/routers/index.ts`
3. Use in renderer: `trpc.myFeature.myProcedure.useQuery()`

**Add a new tool renderer:**
1. Create component: `src/renderer/features/agents/ui/agent-my-tool.tsx`
2. Register in: `src/renderer/features/agents/ui/agent-tool-registry.tsx`
3. Tool name must match Claude SDK tool use name

**Add a new settings tab:**
1. Create tab: `src/renderer/components/dialogs/settings-tabs/my-tab.tsx`
2. Add to dialog: `src/renderer/components/dialogs/agents-settings-dialog.tsx`

**Add database table:**
1. Add schema: `src/main/lib/db/schema/index.ts`
2. Generate migration: `bun run db:generate`
3. Migration auto-runs on next app start

## Key Patterns

- **All backend calls use tRPC** — no raw IPC. Routers in `src/main/lib/trpc/routers/`, client in `src/renderer/lib/trpc.ts`
- **Window-scoped state** — each Electron window maintains its own selected chat via Jotai atoms
- **Claude SDK** — dynamic import of `@anthropic-ai/claude-agent-sdk`. Two modes: "plan" (read-only) and "agent" (full permissions). Session resume via `sessionId` stored in SubChat. Message streaming via tRPC observable (`claude.onMessage`)
- **Worktree isolation** — each chat gets its own git worktree for code changes
- **MCP integration** — Model Context Protocol servers with OAuth authentication (`src/main/lib/mcp-auth.ts`)
- **Multi-window support** — window ID extracted from URL hash/params via `WindowContext`
- **Auto-update** — checks `https://cdn.21st.dev/releases/desktop/latest-mac.yml` on startup and window focus (1 min cooldown)
- **Dev vs Prod isolation** — dev uses `twentyfirst-agents-dev://` protocol and separate userData path (`Agents Dev/`)

## File Naming Conventions

- Components: PascalCase (`ActiveChat.tsx`)
- Utilities/hooks: camelCase (`useFileUpload.ts`, `formatters.ts`)
- Stores: kebab-case (`sub-chat-store.ts`)
- Atoms: camelCase with `Atom` suffix (`selectedAgentChatIdAtom`)

## Build Configuration

- **electron-vite** bundles three targets (main, preload, renderer)
- Main process: CJS output, externalizes better-sqlite3 and claude-agent-sdk
- Renderer: two entry points (`index.html`, `login.html`), path alias `@` → `src/renderer/`
- ASAR with unpacking for: better-sqlite3, node-pty, claude-agent-sdk
- `postinstall` rebuilds native modules for Electron

## Important Gotchas

- `bun run claude:download` must be run before first `bun run dev` — without the Claude binary, agent chat won't function
- **mock-api.ts is DEPRECATED** — Use tRPC directly: `trpc.router.procedure.useQuery()` instead of mock-api wrappers. See `src/renderer/lib/trpc.ts` for client setup.
- DevTools unlock requires 5 clicks on a hidden UI element (not exposed by default)
- Protocol handler registration on macOS may not work on first app launch — users may need to click "Sign in" again
- Analytics (PostHog) and error tracking (Sentry) are disabled in open source builds unless env vars are set in `.env.local`

## Verification

**No automated test suite configured.** Verify changes manually:

```bash
bun run ts:check           # Type validation
bun run build              # Compilation check
bun run dev                # Manual testing in Electron
```

**Critical paths to verify:**
- Create new chat → send message → see response
- Git changes panel → stage files → commit
- Terminal → run command → see output
- Settings → change theme → see update

## Compact Instructions

When compacting, always preserve: the list of modified files, the current task context, any test commands or verification steps, and the tRPC router structure being worked on.
