# Codebase Concerns

**Analysis Date:** 2026-02-01

## Tech Debt

**Missing Test Suite:**
- Issue: Project explicitly has no test suite or linter configured (`CLAUDE.md` states "There is no test suite or linter configured in this project")
- Files: All source files (`src/`)
- Impact: Zero test coverage on critical paths (chat streaming, worktree management, database operations, IPC). Changes cause regressions that go undetected. UI state bugs cascade without validation.
- Fix approach: Establish testing infrastructure with Jest/Vitest, create unit tests for core logic (tRPC routers, message store, git operations), add integration tests for multi-process IPC, implement E2E tests for critical user flows

**Excessive Use of `any` Type:**
- Issue: 30+ instances of `any[]`, `any`, and loose typing throughout codebase, concentrated in message handling and tool rendering
- Files: `src/main/lib/trpc/routers/claude.ts` (line 493, 570, 616), `src/renderer/features/agents/ui/agent-tool-utils.ts`, `src/renderer/features/agents/main/active-chat.tsx` (multiple), `src/preload/index.ts` (line 212), `src/main/lib/vscode-theme-scanner.ts` (line 39)
- Impact: Type safety breaks down in message part transformation, tool state handling, and streaming logic. Refactoring or adding features risks silent failures. Runtime errors possible when message structure changes.
- Fix approach: Create explicit types for MessagePart, ToolState, UIMessageChunk. Use stricter tsconfig settings (noImplicitAny: true). Gradually migrate any[] to typed arrays starting with critical paths.

**Deprecated mock-api.ts:**
- Issue: Entire renderer feature depends on `src/renderer/lib/mock-api.ts` which is marked DEPRECATED in CLAUDE.md but still serves as primary API abstraction
- Files: `src/renderer/lib/mock-api.ts` (16k lines), consumed across all agent features
- Impact: Difficult to remove because it's the central bridge between renderer and tRPC. Any new code still uses it by convention despite deprecation flag. Dead code patterns accumulate.
- Fix approach: Create migration plan: (1) Create direct tRPC hooks layer to replace mock-api wrapper, (2) Incrementally move features to use tRPC directly, (3) Phase out mock-api stubs as features migrate, (4) Remove file in final pass

**Scattered State Management Patterns:**
- Issue: Multiple competing state patterns: Jotai atoms, Zustand stores, React Query, localStorage migrations
- Files: `src/renderer/features/agents/atoms/`, `src/renderer/features/agents/stores/` (message-store.ts, sub-chat-store.ts, streaming-status-store.ts), `src/renderer/lib/atoms/`, `src/renderer/lib/stores/`
- Impact: Complex data flow is hard to trace. Same concept (e.g., open sub-chats) is duplicated across atoms, Zustand, and localStorage. State synchronization bugs possible when one source of truth diverges.
- Fix approach: Create state architecture document. Establish single source of truth per concern. Use Jotai for local UI state, React Query for server state, Zustand only for complex cross-component flows. Remove localStorage management from Zustand (move to custom hook).

**Database Migration Versioning Fragile:**
- Issue: Drizzle migrations in `drizzle/` directory but also duplicated in `resources/migrations/` for packaged app. Version mismatch possible if dev and prod diverge.
- Files: `drizzle/`, `resources/migrations/`, `src/main/lib/db/schema/index.ts`
- Impact: Packaged app may run different schema than development. New migrations added to drizzle/ but forgotten in resources/ causes version skew. Users on old schema can't upgrade gracefully.
- Fix approach: Single-source schema versioning. Build process must copy migrations to resources/ automatically. Add schema version check on startup that fails loudly if migrations don't match.

## Known Bugs

**Sub-Chat State Inconsistency After Tab Close:**
- Symptoms: Switching between chats quickly after closing a sub-chat tab sometimes leaves orphaned message data in Jotai atom store
- Files: `src/renderer/features/agents/stores/sub-chat-store.ts` (removeFromOpenSubChats), `src/renderer/features/agents/main/active-chat.tsx` (tab close handlers)
- Trigger: (1) Open multiple sub-chat tabs, (2) Close one tab, (3) Switch to another chat/back rapidly before store cleanup completes
- Workaround: Refresh app. Data persists in DB but orphaned from Jotai cache.

**Claude Binary Download Required Before First Dev Run:**
- Symptoms: `bun run dev` starts Electron but agent chat doesn't function - no error message shown to user
- Files: `src/main/index.ts` (binary check), postinstall script
- Trigger: Fresh clone or new dev setup without running `bun run claude:download`
- Workaround: Run `bun run claude:download` manually. Should be automatic or checked at startup with user-visible error.

**Protocol Handler Registration May Fail on macOS First Launch:**
- Symptoms: OAuth sign-in via deep link doesn't work after fresh install on macOS
- Files: `src/main/windows/main.ts` (protocol handler setup)
- Trigger: First app launch on macOS where user hasn't configured system to allow the protocol
- Workaround: Click "Sign in" button again. Protocol is registered after first launch completes.

**MCP OAuth Token Refresh Race Condition:**
- Symptoms: Concurrent requests to MCP servers with OAuth may fail if token refresh happens between requests
- Files: `src/main/lib/mcp-auth.ts` (ensureMcpTokensFresh), `src/main/lib/trpc/routers/claude.ts` (MCP server initialization)
- Trigger: Multiple message streams accessing same MCP server simultaneously while token expires
- Workaround: Sequential message sending avoids race. Refresh tokens before batch operations.

## Security Considerations

**Tool Name Injection in Claude SDK:**
- Risk: User-provided tool names embedded in prompt could manipulate Claude's tool selection if not validated
- Files: `src/main/lib/trpc/routers/claude.ts` (line 65-75 in parseMentions function)
- Current mitigation: Tool names validated with regex `/^[a-zA-Z0-9_-]+$/` to allow only safe characters
- Recommendations: (1) Add unit tests for parseMentions validation, (2) Whitelist tool names against actual available tools before sending to SDK, (3) Add telemetry for rejected tool mentions to detect attack patterns

**Electron Safe Storage Encryption Optional:**
- Risk: OAuth tokens and credentials encrypted only if safe storage is available; falls back to base64 encoding on unavailable systems
- Files: `src/main/lib/trpc/routers/claude.ts` (decryptToken function), `src/main/auth-store.ts`
- Current mitigation: Tokens stored in SQLite database only readable by app process (OS-level file permissions)
- Recommendations: (1) Log warning when encryption unavailable, (2) Require encrypted storage on production builds, (3) Add periodic token rotation as additional defense

**Unencrypted Credentials in Debug Settings:**
- Risk: Debug tab allows viewing system info and credentials via file system access
- Files: `src/renderer/components/dialogs/settings-tabs/agents-debug-tab.tsx`
- Current mitigation: Requires intentional UI navigation
- Recommendations: (1) Add confirmation dialog before exposing sensitive paths, (2) Mask API keys/tokens in debug output, (3) Log all debug tab access attempts

**MCP Server Config Exposure via ~/.claude.json:**
- Risk: All MCP server configurations including OAuth tokens stored in plaintext ~/.claude.json
- Files: `src/main/lib/claude-config.ts`, warmup logic reads config from home directory
- Current mitigation: File permissions depend on OS (typically user-readable)
- Recommendations: (1) Encrypt MCP credentials when writing config, (2) Document that ~/.claude.json must be kept private, (3) Add warning if file permissions too open (644 instead of 600)

## Performance Bottlenecks

**Large Component Re-renders on Message Stream:**
- Problem: active-chat.tsx is 6000+ lines. When messages stream in, entire component tree potentially re-renders
- Files: `src/renderer/features/agents/main/active-chat.tsx` (233k+ chars), renders message list, diff view, plan sidebar simultaneously
- Cause: Jotai atoms at top level. When messageAtomFamily updates, all derived atoms recalculate, all consumers re-render
- Improvement path: (1) Extract message list into isolated component with memo boundaries, (2) Use messageStructureAtomFamily to separate structure from text content streaming, (3) Lazy-load diff view and plan sidebar (not needed during message streaming), (4) Profile with React DevTools to identify exact re-render paths

**Excessive JSON Serialization of Messages:**
- Problem: Messages stored in DB as JSON strings, parsed on every retrieve, re-serialized to tRPC, parsed in renderer, passed through message store atoms
- Files: `src/main/lib/db/schema/index.ts` (sub_chats.messages stored as TEXT), mock-api.ts (line 50-80 JSON.parse), message stores
- Cause: Database schema uses TEXT for flexibility during development, never optimized
- Improvement path: (1) Benchmark JSON parse/stringify time for typical 100-message chat, (2) Consider storing parts as separate table rows (normalized schema), (3) Add caching layer for parsed messages in renderer (LRU cache for last 50 messages), (4) Compress old messages before storage

**MCP Server Warmup on Startup Blocks App:**
- Problem: warmupMcpCache() iterates all projects and issues "ping" queries to all MCP servers synchronously at app startup
- Files: `src/main/lib/trpc/routers/claude.ts` (warmupMcpCache function, ~50-100ms per server)
- Cause: App waits for warmup to complete before renderer process starts
- Improvement path: (1) Run warmup asynchronously after renderer loads, (2) Cache MCP server statuses to disk, (3) Only warmup servers for active project (not all projects), (4) Add startup telemetry to measure impact

**Terminal Rendering Lag with Large Output:**
- Problem: xterm.js rendering 10k+ lines of output causes UI freezes
- Files: `src/renderer/features/terminal/terminal.tsx`
- Cause: Terminal renders all lines in DOM, searches entire buffer for link detection
- Improvement path: (1) Implement virtual scrolling (only render visible lines), (2) Debounce link detection during rapid writes, (3) Move link detection to worker thread, (4) Add settings to limit terminal buffer size

## Fragile Areas

**Message Store Atom Family Cleanup Incomplete:**
- Files: `src/renderer/features/agents/stores/message-store.ts` (messageAtomFamily, no cleanup function), `src/renderer/features/agents/main/active-chat.tsx`
- Why fragile: When switching chats, old message atoms remain in memory. If user has many chats, memory usage grows unbounded. Atom family doesn't have automatic cleanup.
- Safe modification: (1) Add explicit cleanup function that removes atoms for closed chats, (2) Track activeMessageIdsByChat and delete atoms not in active set, (3) Call cleanup in chat.unsubscribe() or window.beforeunload
- Test coverage: No tests for memory leaks during chat switching

**Git Worktree Creation Race Condition:**
- Files: `src/main/lib/git/` (worktree initialization), `src/main/lib/trpc/routers/chats.ts`
- Why fragile: Concurrent message handlers may attempt to create same worktree simultaneously. `git worktree add` fails if path exists.
- Safe modification: (1) Add mutex lock per chat before worktree creation, (2) Check if worktree exists before attempting create, (3) Handle EEXIST gracefully with retry or skip
- Test coverage: No tests for concurrent chat operations

**Diff View Large File Rendering:**
- Files: `src/renderer/features/agents/ui/agent-diff-view.tsx` (skipHighlightingForLargeFiles check)
- Why fragile: Comment says "skip diff metadata" but actual file loading may still parse entire file into memory for comparison
- Safe modification: (1) Add explicit size check before parsing ANY file content, (2) Warn user if diff larger than 1MB, (3) Implement chunked diff parsing for large files
- Test coverage: No tests for files >10MB

**Sub-Chat Message Sync Vulnerability:**
- Files: `src/main/lib/trpc/routers/chats.ts` (handleMessageSync), message store synchronization
- Why fragile: If network drops during message sync, some parts recorded in DB but not in Jotai store. Subsequent operations work with stale state.
- Safe modification: (1) Add transaction wrapper for DB write + atom update, (2) Retry failed syncs with exponential backoff, (3) Add consistency check on chat open: compare DB messages count vs Jotai count
- Test coverage: No tests for network failure scenarios

## Scaling Limits

**Single SQLite Connection Bottleneck:**
- Current capacity: ~100 concurrent message operations before connection queue backs up
- Limit: SQLite with better-sqlite3 is synchronous. Main process thread blocks during DB writes.
- Scaling path: (1) Implement write-ahead log (WAL) mode properly (already enabled but verify journal_mode), (2) Move heavy queries to worker thread, (3) Consider PostgreSQL for true multi-client architecture, (4) Profile DB operations to find slow queries

**Terminal Emulation Memory Growth:**
- Current capacity: 100k lines of output before noticeable lag
- Limit: xterm.js keeps all lines in memory for scrollback
- Scaling path: (1) Implement circular buffer (drop oldest lines above threshold), (2) Add setting for max buffer size, (3) Move terminal state to separate process if multiple terminals open, (4) Compress historical terminal sessions to disk

**Chat History Growth Unbounded:**
- Current capacity: 1000+ messages per chat before message list scroll becomes slow
- Limit: All messages loaded into Jotai atoms on chat open
- Scaling path: (1) Paginate messages (load newest 100, load older on scroll), (2) Archive old messages to separate table, (3) Add pagination UI for "Load earlier messages", (4) Implement efficient message search index

## Dependencies at Risk

**@anthropic-ai/claude-agent-sdk Dynamic Import:**
- Risk: SDK imported dynamically and cached globally. Version mismatch if updated without app restart.
- Impact: Updates to SDK require app restart. No hot-reload possible.
- Migration plan: (1) Document SDK version pinning requirements, (2) Add startup check for SDK version, (3) If major version mismatch detected, prompt user to restart app

**better-sqlite3 Native Module:**
- Risk: Requires rebuild for each Electron version. Postinstall script rebuilds, but can fail silently on some systems.
- Impact: Build failure on fresh clone not caught until `bun run dev` attempts database access.
- Migration plan: (1) Add explicit check after postinstall confirming rebuild succeeded, (2) Pre-built binaries for common platforms, (3) Fallback to HTTP-based remote database if native module unavailable

**node-pty Native Module:**
- Risk: Terminal ceases functioning if module doesn't rebuild for target Electron version.
- Impact: Terminal feature silently broken on some builds.
- Migration plan: (1) Wrap node-pty in try-catch with user-visible error, (2) Disable terminal UI if module unavailable, (3) Test builds explicitly verify both native modules load

**electron-vite Build Chain:**
- Risk: Complex bundling config in electron.vite.config.ts. Changes to main/renderer boundary can break IPC.
- Impact: Refactoring main process requires understanding three build targets (main, preload, renderer).
- Migration plan: (1) Create build validation tests (verify preload exposes expected APIs), (2) Document IPC contract separately, (3) Add build-time checks for undefined exports

## Missing Critical Features

**End-to-End Testing for Chat Workflows:**
- Problem: No E2E tests for create-chat → send-message → see-result flow
- Blocks: Can't safely refactor streaming logic, can't verify OAuth works end-to-end, can't test multi-window scenarios
- Impact: Regressions go undetected. Users hit bugs that would fail E2E test.

**Local Fallback When Claude API Unavailable:**
- Problem: If API down, user sees error with no recourse. No built-in fallback to offline mode or alternative model.
- Blocks: Users cannot work when API is temporarily unavailable.
- Current: Ollama integration exists but requires manual model download, not suggested in UI.

**Conflict Resolution UI for Worktree Divergence:**
- Problem: If main branch advances while worktree is open, merge conflicts can occur with no UI to resolve.
- Blocks: Users can't merge changes safely, worktree becomes unusable.
- Current: Some handling exists but not exposed to user.

**Message History Search:**
- Problem: No way to search within a chat's message history.
- Blocks: Long chats become hard to navigate.
- Current: Chat search bar exists but uses simple substring matching, no indexing.

## Test Coverage Gaps

**Message Streaming and State Synchronization:**
- What's not tested: Multi-part streaming messages, out-of-order part arrival, tool state transitions during streaming
- Files: `src/main/lib/trpc/routers/claude.ts` (entire observable logic), `src/renderer/features/agents/stores/message-store.ts`
- Risk: Streaming bugs are frequent cause of lost/duplicate messages in production
- Priority: High - affects core user workflow

**tRPC Preload Bridge:**
- What's not tested: IPC message serialization, window ID isolation, error propagation through preload
- Files: `src/preload/index.ts`, `src/renderer/lib/trpc.ts`
- Risk: Multi-window state leakage, type mismatches between processes
- Priority: High - affects multi-window feature stability

**Git Worktree Operations:**
- What's not tested: Concurrent worktree creation, cleanup on error, stash creation/restoration
- Files: `src/main/lib/git/`, worktree initialization
- Risk: Worktree data corruption, cleanup failing silently
- Priority: High - data loss risk

**Auth Token Refresh:**
- What's not tested: Token expiry during message, refresh timeout, concurrent auth requests
- Files: `src/main/auth-manager.ts`, `src/main/lib/trpc/routers/claude.ts`
- Risk: Auth failures mid-stream, tokens used after expiry
- Priority: Medium - affects reliability

**MCP Server Initialization:**
- What's not tested: MCP server connection failures, auth failures, server crashes during operation
- Files: `src/main/lib/mcp-auth.ts`, `src/main/lib/trpc/routers/claude.ts` (MCP initialization)
- Risk: Silent MCP failures, tools unavailable with no error feedback
- Priority: Medium - affects feature availability

**Multi-Window Isolation:**
- What's not tested: Window-scoped state isolation, atom sharing between windows, localStorage collisions
- Files: Window context, Jotai atoms, localStorage management
- Risk: State leakage between windows, simultaneous chats interfering
- Priority: Medium - affects multi-window feature correctness

---

*Concerns audit: 2026-02-01*
