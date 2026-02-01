# Codebase Structure

**Analysis Date:** 2026-02-01

## Directory Layout

```
1code/
├── src/
│   ├── main/                          # Electron main process
│   │   ├── index.ts                   # App entry, lifecycle, protocol handlers
│   │   ├── auth-manager.ts            # OAuth token exchange and refresh
│   │   ├── auth-store.ts              # Encrypted credential storage
│   │   ├── constants.ts               # AUTH_SERVER_PORT, PROTOCOL, IS_DEV
│   │   ├── windows/                   # Window management
│   │   │   ├── main.ts                # Window creation and IPC handlers
│   │   │   └── window-manager.ts      # WindowManager singleton (stable IDs, lifecycle)
│   │   └── lib/
│   │       ├── db/                    # Database layer
│   │       │   ├── index.ts           # initDatabase(), closeDatabase()
│   │       │   ├── schema/index.ts    # Drizzle table definitions (projects, chats, sub_chats, credentials)
│   │       │   └── utils.ts           # createId(), helpers
│   │       ├── claude/                # Claude SDK integration
│   │       │   ├── index.ts           # Exports
│   │       │   ├── env.ts             # buildClaudeEnv(), shell environment
│   │       │   ├── transform.ts       # Transform raw Claude messages to UI format
│   │       │   ├── raw-logger.ts      # Log raw Claude messages to disk
│   │       │   ├── offline-handler.ts # Offline fallback detection
│   │       │   └── types.ts           # UIMessageChunk, MessageMetadata types
│   │       ├── git/                   # Git operations and file watching
│   │       │   ├── index.ts           # Exports
│   │       │   ├── git-factory.ts     # createSimpleGit() with security
│   │       │   ├── git-operations.ts  # Commit, stash, branch operations
│   │       │   ├── worktree.ts        # createWorktree(), cleanupWorktrees()
│   │       │   ├── worktree-naming.ts # Generate stable worktree names
│   │       │   ├── staging.ts         # Stage/unstage files
│   │       │   ├── stash.ts           # Stash operations
│   │       │   ├── status.ts          # Git status parsing
│   │       │   ├── branches.ts        # List branches, get current branch
│   │       │   ├── file-contents.ts   # Read file from git trees
│   │       │   ├── diff-parser.ts     # Parse git diffs
│   │       │   ├── github/            # GitHub API integration
│   │       │   │   ├── github.ts      # PR creation, OAuth, Octokit client
│   │       │   │   └── types.ts       # GitHub types
│   │       │   ├── security/          # Path validation and command escaping
│   │       │   │   ├── git-commands.ts # Safe git command execution
│   │       │   │   ├── path-validation.ts
│   │       │   │   └── secure-fs.ts   # Filesystem safety checks
│   │       │   ├── cache/             # Git operation caching
│   │       │   │   └── git-cache.ts
│   │       │   ├── watcher/           # File system watcher for git changes
│   │       │   │   ├── git-watcher.ts # Chokidar-based file watcher
│   │       │   │   └── ipc-bridge.ts  # Send git status to renderer
│   │       │   ├── utils/             # Utility functions
│   │       │   │   ├── parse-status.ts
│   │       │   │   └── apply-numstat.ts
│   │       │   └── offline-utils.ts   # Offline operations
│   │       ├── terminal/              # Terminal management via node-pty
│   │       │   ├── index.ts           # TerminalManager singleton
│   │       │   ├── manager.ts         # Terminal session lifecycle
│   │       │   ├── session.ts         # Individual terminal session
│   │       │   ├── env.ts             # Environment setup
│   │       │   ├── data-batcher.ts    # Batch terminal output for performance
│   │       │   ├── port-scanner.ts    # Find available ports
│   │       │   ├── port-manager.ts    # Track allocated ports
│   │       │   ├── terminal-escape-filter.ts # ANSI escape handling
│   │       │   ├── types.ts           # TerminalSession type
│   │       │   └── terminal-history/  # Terminal command history
│   │       ├── trpc/                  # tRPC server setup
│   │       │   ├── index.ts           # tRPC context, router, middleware
│   │       │   └── routers/           # 15 tRPC routers
│   │       │       ├── index.ts       # Merge all routers into appRouter
│   │       │       ├── projects.ts    # Project CRUD, git remote extraction
│   │       │       ├── chats.ts       # Chat CRUD, worktree creation
│   │       │       ├── claude.ts      # Claude streaming observable
│   │       │       ├── claude-code.ts # Claude Code OAuth, CLI config check
│   │       │       ├── terminal.ts    # Terminal create, execute, kill
│   │       │       ├── files.ts       # File read/write, upload handler
│   │       │       ├── agents.ts      # Agent query procedures
│   │       │       ├── agent-utils.ts # Shared agent logic
│   │       │       ├── skills.ts      # Skill CRUD
│   │       │       ├── commands.ts    # Custom command CRUD
│   │       │       ├── ollama.ts      # Ollama model detection
│   │       │       ├── external.ts    # External service calls (web search, fetch)
│   │       │       ├── debug.ts       # Debug utilities
│   │       │       ├── voice.ts       # Voice transcription
│   │       │       └── worktree-config.ts # Worktree configuration
│   │       ├── auth-manager.ts        # OAuth flow, token management
│   │       ├── credential-manager.ts  # Credential storage/retrieval
│   │       ├── auto-updater.ts        # electron-updater integration
│   │       ├── analytics.ts           # PostHog client
│   │       ├── mcp-auth.ts            # MCP OAuth callback handling
│   │       ├── oauth.ts               # OAuth utilities
│   │       ├── claude-config.ts       # Claude configuration
│   │       ├── claude-token.ts        # Token file management
│   │       ├── cli.ts                 # CLI binary management (install/uninstall)
│   │       ├── config.ts              # Config file utilities
│   │       ├── window.ts              # Window control helpers
│   │       ├── vscode-theme-scanner.ts # VS Code theme discovery
│   │       └── ...other utilities
│   │
│   ├── preload/
│   │   └── index.ts                   # Context-isolated API bridge
│   │       - Exposes `window.desktopApi` (window controls, auth, file watchers)
│   │       - Exposes tRPC bridge via `exposeElectronTRPC()`
│   │       - Sentry initialization (production only)
│   │
│   ├── renderer/                       # React 19 UI
│   │   ├── App.tsx                    # Root component, onboarding router
│   │   ├── index.html                 # Main entry point
│   │   ├── login.html                 # Login page entry point
│   │   ├── contexts/
│   │   │   ├── TRPCProvider.tsx        # tRPC client setup with QueryClientProvider
│   │   │   └── WindowContext.tsx       # Window ID and params extraction
│   │   ├── components/
│   │   │   ├── dialogs/               # Modal dialogs
│   │   │   │   ├── agents-settings-dialog.tsx # Settings modal
│   │   │   │   ├── claude-login-modal.tsx
│   │   │   │   └── settings-tabs/     # Individual setting tabs
│   │   │   │       ├── agents-profile-tab.tsx
│   │   │   │       ├── agents-appearance-tab.tsx
│   │   │   │       ├── agents-models-tab.tsx
│   │   │   │       ├── agents-mcp-tab.tsx
│   │   │   │       ├── agents-skills-tab.tsx
│   │   │   │       ├── agents-keyboard-tab.tsx
│   │   │   │       └── ...other tabs
│   │   │   ├── ui/                    # shadcn/ui components + custom
│   │   │   │   ├── button.tsx
│   │   │   │   ├── dialog.tsx
│   │   │   │   ├── resizable-sidebar.tsx # Draggable sidebar borders
│   │   │   │   ├── prompt-input.tsx   # Rich text input for messages
│   │   │   │   ├── icons.tsx          # Icon library
│   │   │   │   └── ...other components
│   │   │   ├── chat-markdown-renderer.tsx
│   │   │   ├── update-banner.tsx
│   │   │   ├── windows-title-bar.tsx
│   │   │   └── ...other components
│   │   ├── features/
│   │   │   ├── agents/                # Core chat interface
│   │   │   │   ├── main/              # Main chat views
│   │   │   │   │   ├── active-chat.tsx # Chat view (huge - streams, tool rendering, UI)
│   │   │   │   │   ├── chat-data-sync.tsx # Fetch sub-chats and messages
│   │   │   │   │   ├── chat-input-area.tsx # Message input controls
│   │   │   │   │   ├── new-chat-form.tsx # New chat creation
│   │   │   │   │   ├── messages-list.tsx # Message rendering
│   │   │   │   │   ├── assistant-message-item.tsx # Individual assistant message
│   │   │   │   │   ├── isolated-messages-section.tsx # Message groups
│   │   │   │   │   └── ...other message components
│   │   │   │   ├── ui/                # Tool renderers (40+ components)
│   │   │   │   │   ├── agent-tool-registry.tsx # Tool type → component mapping
│   │   │   │   │   ├── agent-tool-call.tsx # Generic tool wrapper
│   │   │   │   │   ├── agent-bash-tool.tsx # Bash output
│   │   │   │   │   ├── agent-edit-tool.tsx # File edits
│   │   │   │   │   ├── agent-diff-view.tsx # Diff renderer
│   │   │   │   │   ├── agent-plan-tool.tsx # Plan output
│   │   │   │   │   ├── agent-preview.tsx # Code preview window
│   │   │   │   │   ├── agents-header-controls.tsx # Header buttons
│   │   │   │   │   └── ...other tool renderers
│   │   │   │   ├── atoms/             # Jotai atoms (window-scoped)
│   │   │   │   │   └── index.ts       # selectedAgentChatIdAtom, selectedProjectAtom, etc.
│   │   │   │   ├── stores/            # Zustand stores (complex state)
│   │   │   │   │   ├── sub-chat-store.ts # Open tabs, active tab, pinned tabs (localStorage + migration)
│   │   │   │   │   ├── message-store.ts # Message list per sub-chat
│   │   │   │   │   ├── message-queue-store.ts # Pending messages to send
│   │   │   │   │   └── streaming-status-store.ts # Active streams per sub-chat
│   │   │   │   ├── commands/          # Slash command definitions
│   │   │   │   │   ├── builtin-commands.ts # /help, /clear, etc.
│   │   │   │   │   └── agents-slash-command.tsx
│   │   │   │   ├── mentions/          # File mention system
│   │   │   │   │   ├── agents-mentions-editor.tsx # Mention input autocomplete
│   │   │   │   │   └── render-file-mentions.tsx # Render @file references
│   │   │   │   ├── search/            # Chat search
│   │   │   │   │   ├── chat-search-bar.tsx
│   │   │   │   │   └── chat-search-utils.ts
│   │   │   │   ├── hooks/             # Custom hooks
│   │   │   │   │   ├── use-agents-file-upload.ts # Drag-drop file handling
│   │   │   │   │   ├── use-changed-files-tracking.ts # Track modified files
│   │   │   │   │   ├── use-desktop-notifications.ts # Native notifications
│   │   │   │   │   ├── use-text-context-selection.ts # Text selection
│   │   │   │   │   └── ...other hooks
│   │   │   │   ├── lib/               # Shared utilities
│   │   │   │   │   ├── ipc-chat-transport.ts # tRPC transport for chat streaming
│   │   │   │   │   ├── queue-utils.ts # Queue item creation helpers
│   │   │   │   │   ├── drafts.ts      # Draft message persistence
│   │   │   │   │   ├── agents-actions.ts # UI action handlers
│   │   │   │   │   └── ...other utilities
│   │   │   │   ├── components/        # Sub-components
│   │   │   │   │   ├── agent-send-button.tsx
│   │   │   │   │   ├── queue-processor.tsx # Process message queue
│   │   │   │   │   ├── agents-quick-switch-dialog.tsx # Chat switcher (Ctrl+Tab)
│   │   │   │   │   └── ...other components
│   │   │   │   ├── context/           # React context
│   │   │   │   │   └── text-selection-context.tsx
│   │   │   │   └── constants.ts
│   │   │   ├── sidebar/               # Left sidebar
│   │   │   │   ├── agents-sidebar.tsx # Chat list, new chat button
│   │   │   │   ├── agents-subchats-sidebar.tsx # Sub-chat tabs
│   │   │   │   └── ...utilities
│   │   │   ├── changes/               # Git changes panel
│   │   │   │   ├── changes/           # Main changes view
│   │   │   │   ├── components/        # File list, diff, commit
│   │   │   │   │   ├── file-item.tsx
│   │   │   │   │   ├── file-list.tsx
│   │   │   │   │   ├── diff-full-page-view.tsx
│   │   │   │   │   ├── diff-sidebar-header.tsx
│   │   │   │   │   ├── commit-input.tsx
│   │   │   │   │   └── ...other components
│   │   │   │   └── utils/             # Git status parsing
│   │   │   ├── terminal/              # Terminal panel
│   │   │   │   ├── terminal-sidebar.tsx # xterm.js wrapper
│   │   │   │   ├── link-providers/    # Link detection in terminal output
│   │   │   │   └── atoms.ts           # Terminal sidebar state per chat
│   │   │   ├── details-sidebar/       # Right sidebar (changes, diff, terminal, plan)
│   │   │   │   ├── details-sidebar.tsx
│   │   │   │   ├── sections/          # Collapsible sections
│   │   │   │   └── atoms.ts           # Panel open/close state
│   │   │   ├── layout/                # Page layout
│   │   │   │   └── agents-layout.tsx  # Main layout with sidebars
│   │   │   ├── onboarding/            # Onboarding flow
│   │   │   │   ├── anthropic-onboarding-page.tsx
│   │   │   │   ├── api-key-onboarding-page.tsx
│   │   │   │   ├── billing-method-page.tsx
│   │   │   │   └── select-repo-page.tsx
│   │   │   ├── kanban/                # Beta kanban view
│   │   │   └── mentions/              # Mentions system
│   │   ├── lib/
│   │   │   ├── trpc.ts                # tRPC client + hooks setup
│   │   │   ├── atoms/                 # Global Jotai atoms (cross-feature)
│   │   │   │   └── index.ts           # Theme, sidebar width, settings dialog, etc.
│   │   │   ├── stores/                # Global Zustand stores
│   │   │   │   └── sub-chat-store.ts  # Re-exported from features
│   │   │   ├── hooks/                 # Global React hooks
│   │   │   │   ├── use-mobile.ts      # Media query hook
│   │   │   │   ├── use-file-change-listener.ts # Git watcher subscription
│   │   │   │   ├── use-update-checker.ts # Check for updates
│   │   │   │   └── ...other hooks
│   │   │   ├── themes/                # Theme system
│   │   │   │   └── theme-provider.tsx # VS Code theme integration
│   │   │   ├── utils/
│   │   │   │   ├── platform.ts        # isDesktopApp(), platform detection
│   │   │   │   ├── formatters.ts      # Number, date formatting
│   │   │   │   └── ...other utilities
│   │   │   ├── hotkeys/               # Keyboard shortcuts
│   │   │   ├── jotai-store.ts         # Jotai store instance
│   │   │   ├── analytics.ts           # PostHog client
│   │   │   ├── api-fetch.ts           # Fetch wrapper
│   │   │   └── mock-api.ts            # Mock API (deprecated)
│   │   ├── public/                    # Static assets
│   │   ├── styles/                    # Global CSS
│   │   │   └── globals.css
│   │   └── icons/                     # Custom icon components
│   │
│   └── shared/
│       └── changes-types.ts           # Shared type definitions
│
├── drizzle/                           # Database migrations (dev)
├── resources/                         # Resources for packaging
│   ├── migrations/                    # Migrations (packaged)
│   └── ...icons and resources
├── assets/                            # App assets
├── build/                             # Build output
├── out/                               # Electron build output
├── electron.vite.config.ts            # Build configuration
├── tsconfig.json                      # TypeScript config
├── tailwind.config.js                 # Tailwind CSS config
└── package.json                       # Dependencies and scripts
```

## Directory Purposes

**src/main/:**
- Purpose: All Electron main process code
- Contains: App lifecycle, database, git, terminal, tRPC routers, auth
- Key files: `index.ts` (entry point), `windows/main.ts` (window creation)

**src/preload/:**
- Purpose: Context-isolated bridge between main and renderer
- Contains: `window.desktopApi` API definitions, tRPC bridge setup
- Key files: `index.ts` (single entry point)

**src/renderer/:**
- Purpose: React UI code
- Contains: Components, hooks, state management (Jotai + Zustand), tRPC client
- Key files: `App.tsx` (root), `features/agents/` (core chat feature)

**src/main/lib/db/:**
- Purpose: Database persistence layer
- Contains: Drizzle ORM schema, initialization
- Key files: `schema/index.ts` (table definitions), `index.ts` (initDatabase)

**src/main/lib/trpc/routers/:**
- Purpose: tRPC procedure definitions
- Contains: 15 routers for different domains
- Key files: `index.ts` (merge all), `claude.ts` (streaming), `projects.ts` (CRUD)

**src/main/lib/git/:**
- Purpose: Git operations and file watching
- Contains: Worktree management, status parsing, diff parsing, file watcher
- Key files: `git-operations.ts`, `worktree.ts`, `watcher/git-watcher.ts`

**src/main/lib/terminal/:**
- Purpose: Terminal session management
- Contains: node-pty wrapper, port management, ANSI handling
- Key files: `manager.ts`, `session.ts`, `data-batcher.ts`

**src/main/lib/claude/:**
- Purpose: Claude SDK integration
- Contains: Environment setup, message transformation, raw logging
- Key files: `env.ts`, `transform.ts`, `raw-logger.ts`

**src/renderer/features/agents/:**
- Purpose: Core chat interface
- Contains: Message list, tool renderers, input area, sub-chat management
- Key files: `main/active-chat.tsx` (huge), `stores/sub-chat-store.ts`, `ui/agent-tool-registry.tsx`

**src/renderer/features/changes/:**
- Purpose: Git changes UI
- Contains: File list, diff viewer, staging, commit
- Key files: `components/file-list.tsx`, `components/diff-full-page-view.tsx`

## Key File Locations

**Entry Points:**
- `src/main/index.ts`: Electron app startup
- `src/renderer/App.tsx`: React app root
- `src/preload/index.ts`: IPC bridge

**Configuration:**
- `package.json`: Dependencies, scripts
- `tsconfig.json`: TypeScript settings
- `electron.vite.config.ts`: Build configuration
- `tailwind.config.js`: Tailwind CSS settings

**Core Logic:**
- `src/main/lib/trpc/routers/claude.ts`: Claude streaming (message I/O)
- `src/main/lib/git/worktree.ts`: Worktree lifecycle
- `src/renderer/features/agents/main/active-chat.tsx`: Chat UI rendering
- `src/renderer/features/agents/stores/sub-chat-store.ts`: Sub-chat tab management

**Testing:**
- No test suite configured (mentioned in CLAUDE.md)

## Naming Conventions

**Files:**
- Components: PascalCase (`ActiveChat.tsx`, `AgentBashTool.tsx`)
- Utilities/hooks: camelCase (`useFileUpload.ts`, `formatters.ts`, `git-operations.ts`)
- Stores: kebab-case (`sub-chat-store.ts`, `message-queue-store.ts`)
- Atoms: camelCase with `Atom` suffix (`selectedAgentChatIdAtom`, `isPlanModeAtom`)

**Directories:**
- Features: kebab-case, feature-scoped (`agents/`, `changes/`, `terminal/`)
- Internal structure: flat or grouped by concern (`stores/`, `ui/`, `hooks/`, `lib/`, `components/`)
- Utilities: kebab-case (`git-operations.ts`, `window-manager.ts`)

**TypeScript:**
- Interfaces: PascalCase (`Context`, `DesktopApi`, `SubChatMeta`)
- Types: PascalCase (`FileStatus`, `UpdateInfo`, `PlayButtonState`)
- Enums: PascalCase, value-based (not string-based when possible)

## Where to Add New Code

**New Feature (major functionality):**
- Primary code: `src/renderer/features/{feature-name}/`
- Main component: `src/renderer/features/{feature-name}/main/{name}.tsx`
- Sub-components: `src/renderer/features/{feature-name}/components/{name}.tsx`
- Utilities: `src/renderer/features/{feature-name}/lib/{name}.ts`
- Tests: `src/renderer/features/{feature-name}/__tests__/{name}.test.ts` (if test suite added)
- Backend: `src/main/lib/trpc/routers/{feature-name}.ts` (new router)

**New Component:**
- If UI-only: `src/renderer/components/ui/{name}.tsx`
- If feature-specific: `src/renderer/features/{feature}/components/{name}.tsx`
- If dialog: `src/renderer/components/dialogs/{name}.tsx`

**New Utility Function:**
- If shared: `src/renderer/lib/utils/{category}.ts` (e.g., `formatters.ts`)
- If main process: `src/main/lib/{category}/{name}.ts` (e.g., `src/main/lib/git/utils.ts`)
- If feature-specific: `src/renderer/features/{feature}/lib/{name}.ts`

**New tRPC Router:**
- File: `src/main/lib/trpc/routers/{name}.ts`
- Export: Add to `src/main/lib/trpc/routers/index.ts` merge
- Type-safe client call: `trpc.{routeName}.{procedureName}.useQuery()` (renderer)

**Global State (Jotai atom):**
- File: `src/renderer/lib/atoms/index.ts` (or feature-specific `atoms/index.ts`)
- Pattern: Use `atomWithStorage()` for persistence
- Window-scoped: Add window ID prefix in localStorage key if needed

**Global State (Zustand store):**
- File: `src/renderer/lib/stores/{name}.ts` or feature-specific `stores/{name}.ts`
- Pattern: Use `create()`, subscribe to changes as needed
- Persistence: Manual localStorage save/load (see `sub-chat-store.ts` pattern)

**Terminal/Git Operation:**
- Main process: `src/main/lib/{terminal|git}/{name}.ts`
- Export via router: `src/main/lib/trpc/routers/{terminal|git}.ts`
- Renderer call: `trpc.{terminal|git}.{operation}.useMutation()`

## Special Directories

**drizzle/:**
- Purpose: Drizzle ORM migrations (generated by `bun run db:generate`)
- Generated: Yes (auto-generated on `db:generate`)
- Committed: Yes (source of truth for schema changes)
- Manual: Do NOT edit migrations directly; edit `src/main/lib/db/schema/index.ts` then regenerate

**resources/:**
- Purpose: Resources bundled with app (migrations, icons)
- Generated: Migrations copied here on build
- Committed: Icons and resources checked in
- Manual: Add native resources here for packaging (app icons, etc.)

**out/:**
- Purpose: Electron build output directory
- Generated: Yes (by `electron-vite`)
- Committed: No (in .gitignore)
- Manual: Never edit

**.planning/codebase/:**
- Purpose: Codebase analysis documents (THIS FILE, ARCHITECTURE.md, STACK.md, etc.)
- Generated: No (manually maintained by Claude agents)
- Committed: Yes
- Manual: Updated by `/gsd:map-codebase` agent runs

