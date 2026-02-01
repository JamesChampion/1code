# Technology Stack

**Analysis Date:** 2026-02-01

## Languages

**Primary:**
- TypeScript 5.4.5 - All source code (main, preload, renderer, utilities)

**Secondary:**
- JavaScript - Build scripts and configuration files

## Runtime

**Environment:**
- Electron 33.4.5 - Desktop application runtime
- Node.js 20.17.50 - Main process runtime

**Package Manager:**
- Bun - Primary package manager (faster than npm)
- Lockfile: `bun.lock` and `bun.lockb` present

## Frameworks

**Core:**
- React 19.2.1 - UI framework for renderer process
- Electron - Desktop application framework

**Desktop/IPC:**
- tRPC 11.7.1 - Type-safe RPC for main↔renderer communication
- trpc-electron 0.1.2 - Electron adapter for tRPC
- electron-log 5.4.3 - Main process logging

**State Management:**
- Jotai 2.11.1 - Atomic state management for renderer (window-scoped state)
- Zustand 5.0.3 - Store management for complex renderer state (sub-chat tabs, message caching)
- React Query 5.90.10 - Server state management (used via tRPC)

**UI Component Library:**
- Radix UI components - Unstyled, accessible UI primitives (`@radix-ui/react-*`)
- Tailwind CSS 3.4.17 - Utility-first CSS styling
- Lucide React 0.468.0 - Icon library
- React Icons 5.5.0 - Additional icon library
- next-themes 0.4.4 - Theme provider (dark mode support)

**Terminal:**
- xterm 5.3.0 - Terminal emulator in renderer
- @xterm/addon-* - Various xterm extensions (fit, search, web-links, canvas, webgl, serialize)
- node-pty 1.1.0 - PTY management for terminal sessions

**AI/LLM:**
- @anthropic-ai/claude-agent-sdk 0.2.12 - Anthropic Claude SDK (dynamic import, ESM)
- ai 6.0.14 - Vercel AI SDK (for streaming support)
- @ai-sdk/react 3.0.14 - React bindings for Vercel AI SDK

**Git & Version Control:**
- simple-git 3.28.0 - Git operations library
- chokidar 5.0.0 - File watcher for project changes
- @git-diff-view/react 0.0.35 - Git diff UI component
- @git-diff-view/shiki 0.0.36 - Syntax highlighting for diffs

**Database & ORM:**
- Drizzle ORM 0.45.1 - Type-safe SQL ORM
- better-sqlite3 11.8.1 - SQLite driver (native module, ASAR unpacked)
- drizzle-kit 0.31.8 - Drizzle schema migrations and tooling

**MCP (Model Context Protocol):**
- @modelcontextprotocol/sdk 1.25.3 - MCP client SDK for tool integration

**Utilities:**
- Zod 3.24.1 - Schema validation and type inference
- superjson 2.2.2 - Serialization for complex types via tRPC
- date-fns 3.6.0 - Date utilities
- clsx 2.1.1 - Conditional classname utility
- class-variance-authority 0.7.1 - Component variant management
- tailwind-merge 2.6.0 - Merge Tailwind classes intelligently
- unique-names-generator 4.7.1 - Generate unique names
- motion 11.15.0 - Animation library
- pidtree 0.6.0 - Process tree utilities (for terminal cleanup)
- react-hotkeys-hook 4.6.1 - Keyboard shortcuts
- react-zoom-pan-pinch 3.7.0 - Interactive zoom/pan
- react-syntax-highlighter 16.1.0 - Code syntax highlighting
- gray-matter 4.0.3 - YAML front-matter parsing
- mermaid 11.12.2 - Diagram rendering (dynamic import)
- streamdown 2.0.1 - Markdown stream parser
- sonner 1.7.1 - Toast notifications
- remark-breaks 4.0.0 - Remark plugin for line breaks
- remark-gfm 4.0.1 - Remark GitHub Flavored Markdown plugin
- shiki 1.24.4 - Code syntax highlighting engine

**Analytics & Error Tracking:**
- @sentry/electron 7.5.0 - Error tracking (optional, disabled unless env var set)
- posthog-js 1.239.1 - Client-side analytics (renderer)
- posthog-node 5.20.0 - Server-side analytics (main process)

**Updates:**
- electron-updater 6.7.3 - Auto-update support
- electron-builder 25.1.8 - App packaging and distribution

## Testing & Development

**Build:**
- electron-vite 3.0.0 - Vite-based build for Electron (bundles main, preload, renderer)
- Vite 6.3.4 - Frontend bundler
- @vitejs/plugin-react 4.3.4 - React plugin for Vite
- TypeScript - Type checking via `tsgo` (see `ts:check` script)

**Linting & Formatting:**
- Not configured - Project lacks ESLint and Prettier

**Type Checking:**
- TypeScript 5.4.5 - Strict mode enabled
- tsgo - External CLI for type checking (invoked via `bun run ts:check`)

## Configuration

**Environment:**
- Vite env vars with `MAIN_VITE_` prefix for main process
- Vite env vars with `VITE_` prefix for renderer
- `.env.example` documents optional vars for analytics, error tracking, APIs, code signing
- `.env.local` files supported for development overrides

**Build:**
- `electron.vite.config.ts` - Electron and Vite configuration
  - Main process: CJS output, externalizes `better-sqlite3`, `claude-agent-sdk`, `@prisma/client`
  - Preload: CJS output, context-isolated
  - Renderer: Two entry points (`index.html`, `login.html`), path alias `@` → `src/renderer/`
- `tsconfig.json` - TypeScript strict mode, ES2022 target
- `postcss.config.js` - Tailwind + Autoprefixer
- `tailwind.config.js` - Custom colors, dark mode, typography plugins
- `electron-builder.yml` - App signing and notarization (macOS)
- `drizzle.config.ts` - SQLite ORM schema location

## Platform Requirements

**Development:**
- Node.js 20.17+ (or Bun runtime equivalent)
- Bun 1.x or npm 10.x
- Python 3.x (for better-sqlite3 rebuild)
- Xcode command-line tools (macOS)

**Production:**
- macOS 10.13+ (Intel and Apple Silicon)
- Windows 7+ (NSIS installer)
- Linux (AppImage, DEB)
- Electron requires unpacking ASAR for native modules: `better-sqlite3`, `node-pty`, `@anthropic-ai/claude-agent-sdk`

## Key Build Artifacts

**ASAR Unpacking:**
- `node_modules/better-sqlite3/**/*` - SQLite native module
- `node_modules/node-pty/**/*` - PTY native module
- `node_modules/@anthropic-ai/claude-agent-sdk/**/*` - Claude SDK (ESM, can't be bundled)

**Resource Embedding:**
- Drizzle migrations: `drizzle/` → packaged as `resources/migrations`
- Platform-specific binaries: `resources/bin/{platform}-{arch}/` → packaged as `bin/`
- VERSION file for release tracking

---

*Stack analysis: 2026-02-01*
