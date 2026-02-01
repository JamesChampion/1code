# External Integrations

**Analysis Date:** 2026-02-01

## APIs & External Services

**21st.dev Backend:**
- Service: Web API at https://21st.dev (production) or custom URL in dev
- Purpose: OAuth authentication, user profile management, subscription plan fetching
- Implementation: `src/main/auth-manager.ts`, `src/main/index.ts`
- Auth: OAuth 2.0 token + refresh token cycle, desktop protocol handler `twentyfirst-agents://` (dev: `twentyfirst-agents-dev://`)
- Env vars: `MAIN_VITE_API_URL` (optional override in dev)
- Endpoints used:
  - `/api/auth/desktop/exchange` - OAuth code exchange
  - `/api/auth/desktop/refresh` - Token refresh (5 min before expiry)
  - `/api/user/profile` - User profile updates
  - `/api/desktop/user/plan` - Fetch subscription plan (for analytics enrichment)
  - `/api/voice/transcribe` - Voice-to-text for authenticated users with paid plans

**OpenAI Whisper API:**
- Service: https://api.openai.com/v1/audio/transcriptions
- Purpose: Voice transcription to text
- Implementation: `src/main/lib/trpc/routers/voice.ts`
- Auth: OpenAI API key (Bearer token)
- Source priority:
  1. User-configured key from settings
  2. Vite env var `MAIN_VITE_OPENAI_API_KEY`
  3. `process.env.OPENAI_API_KEY`
  4. Shell environment (via `execSync`)
- Max audio: 25MB per request
- Timeout: 30 seconds
- Fallback: Uses 21st.dev backend for authenticated users with paid subscriptions if no local key

**Anthropic Claude SDK:**
- Service: Claude API integration
- Purpose: AI agent chat, plan mode execution
- Implementation: `src/main/lib/claude/`, `src/main/lib/trpc/routers/claude.ts`
- Auth: Supports multiple sources:
  - Desktop authentication via OAuth to 21st.dev
  - Custom `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_API_KEY` env var
  - Custom `ANTHROPIC_BASE_URL` for self-hosted/local models
  - OllaMa support for local LLMs
- Dynamic import: ESM module loaded at runtime, not bundled
- SDK features: Agent mode (full execution) and Plan mode (read-only)
- Session resume: Via `sessionId` stored in SubChat table

**GitHub API:**
- Service: GitHub REST API
- Purpose: Fetch PR status, commits, branches
- Implementation: `src/main/lib/git/github/github.ts`
- Auth: GitHub token (from git credential system or user-provided)
- Used for: PR metadata, check status, merge status
- Lazy-loaded: GitHub integration only fetches data when explicitly requested

**Model Context Protocol (MCP):**
- Service: Multiple MCP servers (stdio-based or HTTP)
- Purpose: Tool integration, extensibility
- Implementation: `src/main/lib/mcp-auth.ts`, `src/main/lib/oauth.ts`
- Auth: OAuth 2.0 per server (stored in `~/.claude.json`)
- Client: Official MCP SDK (`@modelcontextprotocol/sdk`)
- Transports:
  - Stdio for local servers
  - HTTP/StreamableHTTP for remote servers
- Token refresh: Automatic 5 minutes before expiry
- Filtered env: Sensitive vars (API keys, secrets) blocked from subprocess environment

## Data Storage

**Databases:**
- SQLite at `{userData}/data/agents.db`
- Connection: better-sqlite3 native driver (no connection pooling)
- Mode: WAL (Write-Ahead Logging) for concurrent access
- Migrations: Auto-run on startup via `initDatabase()` from `drizzle/` or `resources/migrations`
- Schema location: `src/main/lib/db/schema/index.ts`

**ORM:**
- Drizzle ORM 0.45.1
- Query builder: Type-safe SQL
- Migrations: Generated via `drizzle-kit generate`
- Schema source of truth: `src/main/lib/db/schema/index.ts`

**Tables:**
- `projects` - Folders indexed in the app (id, name, path, git metadata)
- `chats` - Workspaces per project (id, name, projectId FK, worktree path, PR tracking)
- `sub_chats` - Sub-conversations within chats (id, name, chatId FK, sessionId, messages JSON, mode)
- `claudeCodeCredentials` - Single-row table for OAuth token storage (encrypted via Electron safeStorage)

**File Storage:**
- Local filesystem only - No cloud storage integration
- User data directory: `{app.getPath('userData')}/data/`
  - Dev: `Agents Dev/` (separate from production)
  - Production: `1Code/`
- Git worktrees: One per chat for code isolation
- Credentials: Encrypted with Electron `safeStorage` API (OS-level encryption)

**Caching:**
- In-memory state via Jotai atoms and Zustand stores
- No distributed cache (Redis, etc.)
- Message cache per sub-chat (JSON in database)
- Voice transcription plan cache (5 minute TTL)

## Authentication & Identity

**Auth Provider:**
- 21st.dev custom OAuth
- Implementation: OAuth 2.0 code exchange flow
- Desktop protocol handler: `twentyfirst-agents://` (prod) or `twentyfirst-agents-dev://` (dev)
- Token storage: Encrypted via Electron `safeStorage` API
- Token refresh: Automatic 5 minutes before expiry (scheduled via timeout)
- Session resumption: Stored in auth store at `{userData}/auth`

**Supported Alt Auth:**
- Custom API key: `ANTHROPIC_API_KEY` or `ANTHROPIC_AUTH_TOKEN` env var (bypasses OAuth)
- OllaMa local LLM: Auto-detected or manually configured

**Multi-account:**
- Single account per desktop session (one OAuth token at a time)
- Logout clears all stored credentials

## Monitoring & Observability

**Error Tracking:**
- Sentry (optional, disabled by default)
- Implementation: `@sentry/electron` 7.5.0
- Env vars:
  - Main: `MAIN_VITE_SENTRY_DSN`
  - Renderer: (uses main process DSN)
- Conditional: Only initialized if env var set and not in dev mode

**Analytics:**
- PostHog (optional, disabled by default in dev)
- Implementation:
  - Main: `posthog-node` 5.20.0 (server-side)
  - Renderer: `posthog-js` 1.239.1 (client-side)
- Env vars:
  - Main: `MAIN_VITE_POSTHOG_KEY`, `MAIN_VITE_POSTHOG_HOST`
  - Renderer: `VITE_POSTHOG_KEY`, `VITE_POSTHOG_HOST`
- Fallback key: Hardcoded public key for open-source analytics if no env var
- Events tracked:
  - `desktop_opened`, `first_launch`, `auth_completed`
  - `project_opened`, `workspace_created`, `workspace_archived`
  - `message_sent`, `pr_created`, `commit_created`, `sub_chat_created`
- User enrichment: Subscription plan, connection method
- Opt-out: User preference synced from renderer to main process

**Logs:**
- Main process: electron-log 5.4.3 to `{userData}/logs/`
- Raw Claude message logging: `src/main/lib/claude/raw-logger.ts` (debug mode)
- Automatic cleanup: Old logs deleted on startup

## CI/CD & Deployment

**Hosting:**
- Electron multi-platform: macOS, Windows, Linux
- Distribution: Generic HTTP provider to CDN
- Update server: `https://cdn.21st.dev/releases/desktop`

**Package Distribution:**
- macOS: DMG + ZIP (arm64 + x64 universal)
- Windows: NSIS installer + Portable EXE
- Linux: AppImage + DEB
- Build config: `electron-builder.yml`

**Code Signing (Optional):**
- macOS: Developer ID code signing + notarization (Apple Gatekeeper)
- Windows: Optional signing (not enforced)
- Env vars:
  - `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID`, `APPLE_IDENTITY`

**Release Pipeline:**
- Command: `bun run release`
- Steps:
  1. Build: TypeScript → Electron (main, preload, renderer)
  2. Download Claude binary: `bun run claude:download`
  3. Package for platforms: `bun run package:mac` (configurable for win/linux)
  4. Generate update manifest: `bun run dist:manifest`
  5. Upload to CDN: `./scripts/upload-release-wrangler.sh`
- Auto-update: Checks for updates on startup and window focus (1 min cooldown)

## Environment Configuration

**Required env vars:**
- No hard requirements - App works without env vars (localhost fallback)

**Optional env vars:**
- `MAIN_VITE_API_URL` - Override 21st.dev backend (dev only)
- `MAIN_VITE_SENTRY_DSN` - Enable Sentry error tracking
- `MAIN_VITE_POSTHOG_KEY`, `MAIN_VITE_POSTHOG_HOST` - Main process analytics
- `VITE_POSTHOG_KEY`, `VITE_POSTHOG_HOST` - Renderer analytics
- `MAIN_VITE_OPENAI_API_KEY` - OpenAI voice transcription key (Vite env)
- `OPENAI_API_KEY` - OpenAI voice key (process.env or shell)
- `ANTHROPIC_AUTH_TOKEN` - Custom Claude auth token
- `ANTHROPIC_API_KEY` - Custom Claude API key
- `ANTHROPIC_BASE_URL` - Custom Claude endpoint (self-hosted)
- `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID`, `APPLE_IDENTITY` - macOS code signing

**Secrets location:**
- `.env.local` files (git-ignored)
- Electron `safeStorage` API for OAuth tokens
- Shell environment (loaded at runtime for app-specific keys)

## Webhooks & Callbacks

**Incoming:**
- OAuth callback: `http://localhost:3001/auth/callback` (dev) or system deep link handler
- MCP OAuth callback: `http://127.0.0.1:3001/callback` (internal redirect URI)
- Auto-update check: GET request to update manifest URL
- GitHub PR status: Fetched on-demand (no webhooks, polling via API)

**Outgoing:**
- 21st.dev auth endpoints: `POST /api/auth/desktop/exchange`, `POST /api/auth/desktop/refresh`
- 21st.dev profile endpoint: `PATCH /api/user/profile`
- OpenAI Whisper: `POST https://api.openai.com/v1/audio/transcriptions`
- GitHub API: `GET` requests for PR metadata
- MCP server calls: All MCP operations via SDK

## Git Integration

**Git Operations:**
- Local git CLI via `simple-git` and direct shell commands
- File watcher: `chokidar` monitors `.git` directory
- Worktree isolation: Each chat gets isolated git worktree
- PR tracking: Stores PR URL and number per chat (manual tracking, no webhooks)

**Security:**
- Path validation: Whitelist for git commands in `src/main/lib/git/security/path-validation.ts`
- Command sanitization: `src/main/lib/git/security/git-commands.ts`
- Filesystem access: Restricted via `src/main/lib/git/security/secure-fs.ts`

---

*Integration audit: 2026-02-01*
