# Architecture

**Analysis Date:** 2026-02-01

## Pattern Overview

**Overall:** Three-process Electron app with tRPC type-safe IPC

**Key Characteristics:**
- Main process (Node.js) handles system lifecycle, database, file operations, Claude SDK integration
- Preload exposes context-isolated `window.desktopApi` and tRPC bridge
- Renderer (React 19 + Jotai + Zustand) provides UI with state management
- tRPC routers handle all backend communication (no raw IPC)
- Window-scoped state via Jotai atoms (each Electron window maintains isolated UI state)
- Worktree isolation per chat (each chat gets separate git worktree for code changes)

## Layers

**Main Process (Electron):**
- Purpose: System lifecycle management, database persistence, tool execution (git, terminal), Claude SDK integration, file I/O
- Location: `src/main/`
- Contains: Electron app lifecycle, tRPC routers, database layer (Drizzle ORM), git operations, terminal management, auth
- Depends on: Electron, SQLite (better-sqlite3), Claude Agent SDK, Node APIs
- Used by: Renderer via tRPC

**Preload (Context-Isolated Bridge):**
- Purpose: Secure communication between main and renderer, exposes safe APIs
- Location: `src/preload/index.ts`
- Contains: tRPC IPC bridge setup, desktopApi object definition, Sentry initialization
- Depends on: trpc-electron, Electron context bridge
- Used by: Renderer code via `window.desktopApi` and Electron tRPC client

**Renderer (React UI):**
- Purpose: User interface with state management and reactive updates
- Location: `src/renderer/`
- Contains: React components, Jotai atoms (window-scoped), Zustand stores (complex state), tRPC client hooks
- Depends on: React 19, Jotai, Zustand, React Query (via tRPC), TailwindCSS, shadcn/ui
- Used by: User interactions

**Database Layer:**
- Purpose: SQLite persistence with automatic migrations
- Location: `src/main/lib/db/`
- Contains: Drizzle ORM schema, migrations, initialization logic
- Depends on: Drizzle ORM, better-sqlite3
- Used by: tRPC routers for data operations

**tRPC Router Layer:**
- Purpose: Type-safe procedure definitions for main-to-renderer communication
- Location: `src/main/lib/trpc/routers/`
- Contains: 15 routers (projects, chats, claude, terminal, git, etc.)
- Depends on: @trpc/server, context (window reference)
- Used by: Renderer via tRPC client

## Data Flow

**Message Streaming (Claude Agent):**

1. Renderer: User submits message via `ActiveChat` component
2. Renderer: `IPCChatTransport` queues message in `useMessageQueueStore` (Zustand store)
3. Renderer: `QueueProcessor` component processes queue items sequentially
4. Main: `claude.onMessage` tRPC observable streams back response chunks
5. Main: Each tool call is streamed as separate chunk (Read, Edit, Bash, etc.)
6. Renderer: Message store accumulates chunks into complete message, renders incrementally
7. Renderer: Tool results are nested within parent tool calls (e.g., file contents under Edit)

**UI State Synchronization:**

1. Jotai atoms (global): `selectedAgentChatIdAtom`, `selectedProjectAtom` (window-scoped via localStorage)
2. Zustand stores (complex): `useAgentSubChatStore` (open tabs, active tab, pinned tabs), `useMessageQueueStore` (pending messages), `useStreamingStatusStore` (active streams)
3. tRPC caching (server state): React Query invalidation triggers refetch from main process
4. localStorage (persistence): Sub-chat tabs restored per window ID (e.g., `main:agent-open-sub-chats-{chatId}`)

**Git Worktree Workflow:**

1. User creates/selects chat
2. Main process creates worktree via `createWorktree()` in chat creation tRPC
3. Git operations isolated to worktree path
4. File watcher monitors worktree for changes (git-status events)
5. Events streamed to renderer via `onGitStatusChanged` listener
6. Changes panel reflects staged/unstaged state

**State Management:**

- **Window-Scoped:** Each Electron window maintains its own selected chat, sub-chats, sidebar state via localStorage key prefixed with window ID (e.g., `main:agent-active-sub-chats-{chatId}`)
- **Atoms (Jotai):** Global reactive state with automatic localStorage persistence (`atomWithStorage`)
- **Stores (Zustand):** Complex state that needs custom subscription logic (message queues, streaming status)
- **tRPC Cache (React Query):** Server state automatically cached, manually invalidated on mutations

## Key Abstractions

**SubChat:**
- Purpose: Independent conversation thread within a chat (tab-like experience)
- Examples: `src/renderer/features/agents/stores/sub-chat-store.ts`, `src/main/lib/db/schema/index.ts`
- Pattern: Each sub-chat maintains its own message history (JSON array in DB), streaming state, and queue. Stores active sub-chat via atom, open sub-chats via Zustand + localStorage

**Chat (Project Context):**
- Purpose: Container for sub-chats scoped to a project with worktree isolation
- Examples: Database `chats` table, `src/main/lib/trpc/routers/chats.ts`
- Pattern: One worktree per chat, branch tracking, PR metadata

**Worktree:**
- Purpose: Isolated git working directory for code changes per chat
- Examples: `src/main/lib/git/worktree.ts`, `src/main/lib/git/git-factory.ts`
- Pattern: Factory creates worktree from main repo, named by chat ID, cleaned up on chat deletion

**tRPC Procedure:**
- Purpose: Type-safe RPC call with error handling
- Examples: `src/main/lib/trpc/routers/claude.ts` (streaming), `src/main/lib/trpc/routers/projects.ts` (queries)
- Pattern: `publicProcedure`, `loggedProcedure` with context containing window reference

**Message Store (Zustand):**
- Purpose: Accumulate streamed message chunks into complete message object
- Examples: `src/renderer/features/agents/stores/message-store.ts`
- Pattern: Per-sub-chat message list, chunk batching, streaming status tracking

## Entry Points

**App Startup (Electron):**
- Location: `src/main/index.ts`
- Triggers: `electron` app ready
- Responsibilities: Initialize auth manager, database, protocol handlers (deep links), create main window, setup IPC handlers, register tRPC observable subscriptions

**Main Window:**
- Location: `src/main/windows/main.ts`
- Triggers: App startup or multi-window create
- Responsibilities: Create BrowserWindow, load HTML/URL, setup window controls, register with WindowManager

**Renderer Entry:**
- Location: `src/renderer/App.tsx`
- Triggers: Window load
- Responsibilities: Initialize analytics, setup auth state check, Jotai store setup, routing to onboarding or main app

**Chat Initialization:**
- Location: `src/renderer/features/agents/main/active-chat.tsx` + `src/renderer/features/agents/main/chat-data-sync.tsx`
- Triggers: Sub-chat selected via atom
- Responsibilities: Initialize message store, setup streaming listener, load message history, bind chat instance to sub-chat ID

## Error Handling

**Strategy:** Async/await with try-catch, error propagated to UI via tRPC error formatting

**Patterns:**
- Main process: Console logging + Sentry (production only)
- Renderer: Sonner toast notifications for user-facing errors
- tRPC: Error messages formatted in middleware, include stack in dev mode
- Claude SDK: Offline fallback handler + raw message logging in `src/main/lib/claude/`

## Cross-Cutting Concerns

**Logging:**
- Main: Console via `console.log` with prefixes (e.g., `[tRPC]`, `[Auth]`, `[Claude]`)
- Claude raw output: Logged to disk via `src/main/lib/claude/raw-logger.ts` for debugging
- Renderer: No persistent logging (in-memory only for dev)

**Validation:**
- Database: Drizzle schema with types (`notNull()`, `unique()` constraints)
- tRPC: Input validation via `input()` on procedures, zod schemas optional
- Git paths: Security validation in `src/main/lib/git/security/` to prevent path traversal

**Authentication:**
- Desktop OAuth: `src/main/auth-manager.ts` exchanges code for token
- Token storage: Encrypted via Electron `safeStorage`
- Cookie-based: Desktop token stored as session cookie in `persist:main` partition
- Auto-refresh: Token refresh 5 min before expiry via `auth-manager.ts`

**Observability:**
- Analytics (PostHog): Opt-in, disabled in open source unless env vars set
- Error tracking (Sentry): Production only, auto-initialized in main + renderer
- Performance: No explicit instrumentation (React DevTools profiler available in dev)

