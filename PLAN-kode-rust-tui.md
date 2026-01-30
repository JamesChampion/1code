# Kode — Rust TUI for Claude Code Workflows

## Overview

A minimal, single-binary Rust TUI that replaces 1Code's Electron shell for personal Linux use. Manages Claude CLI sessions with git worktree isolation, background agent execution, and a vim-like terminal interface.

**Target**: Single Linux workstation (WSL2 or native). No cross-platform, no cloud, no accounts.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    TUI Layer (ratatui)               │
│  ┌───────────┬─────────────────┬──────────────────┐ │
│  │  Sidebar   │   Chat Pane     │  Diff/File View  │ │
│  │  Sessions  │   Messages      │  (toggleable)    │ │
│  │  Status    │   Input         │                  │ │
│  └───────────┴─────────────────┴──────────────────┘ │
├─────────────────────────────────────────────────────┤
│               Application State                      │
│  AppState (shared across TUI + background agents)    │
│  - projects: Vec<Project>                            │
│  - sessions: Vec<Session>                            │
│  - messages per session                              │
│  - active agent handles                              │
├─────────────────────────────────────────────────────┤
│                 Service Layer                        │
│  ┌─────────────┐ ┌────────────┐ ┌────────────────┐ │
│  │ Git/Worktree │ │ Claude CLI │ │  Persistence   │ │
│  │ Manager      │ │ Session    │ │  (SQLite)      │ │
│  │ (git2 + CLI) │ │ (PTY)     │ │  (rusqlite)    │ │
│  └─────────────┘ └────────────┘ └────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Data Flow

1. User input → crossterm `Event` → `App::handle_event()`
2. App dispatches to handler (navigation, text input, command)
3. Handlers mutate `AppState` and/or spawn async operations
4. Claude CLI output streams via `tokio::mpsc` channel → message buffer
5. TUI re-renders current `AppState` each frame (~20 Hz tick)
6. State changes trigger SQLite persistence (on each mutation)

---

## Crate Selection

| Crate | Version | Purpose | Why this one |
|-------|---------|---------|-------------|
| `ratatui` | 0.29 | TUI framework | Active, well-documented, large ecosystem |
| `crossterm` | 0.28 | Terminal backend | Default ratatui backend, Linux-native |
| `tokio` | 1.x | Async runtime | `rt-multi-thread` for background agents |
| `git2` | 0.19 | Git operations | Libgit2 bindings, no CLI dependency for core ops |
| `rusqlite` | 0.32 | SQLite persistence | `bundled` feature, WAL mode, no external dep |
| `serde` | 1.x | Serialization | Standard, used by everything |
| `serde_json` | 1.x | JSON for messages | Message content storage |
| `toml` | 0.8 | Config file format | Human-readable config |
| `anyhow` | 1.x | Error handling | Simple `Result<T>` everywhere |
| `tracing` | 0.1 | Structured logging | File-based log (not stdout, that's TUI) |
| `tracing-subscriber` | 0.3 | Log output | `env-filter` for RUST_LOG control |
| `clap` | 4.x | CLI arg parsing | `derive` feature for zero-boilerplate |
| `uuid` | 1.x | Session IDs | `v4` + `serde` features |
| `chrono` | 0.4 | Timestamps | `serde` feature for DB storage |
| `dirs` | 6.x | XDG directories | Canonical Linux paths |
| `portable-pty` | 0.8 | PTY for Claude CLI | Proper terminal emulation (colors, interactive) |

**Compile profile (release)**: LTO enabled, strip symbols, single codegen unit.

---

## Module Structure

```
kode/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs              # Entry point, tokio runtime, CLI dispatch
│   ├── cli.rs               # Clap arg parsing (init, projects, sessions, default=TUI)
│   ├── config.rs            # XDG paths, TOML config loading, defaults
│   ├── git/
│   │   ├── mod.rs           # Public API re-exports
│   │   └── worktree.rs      # Worktree CRUD (create/list/delete/prune)
│   ├── claude/
│   │   ├── mod.rs           # Public API re-exports
│   │   └── session.rs       # PTY spawn, event streaming, input forwarding
│   ├── state/
│   │   ├── mod.rs           # AppState, domain types (Project, Session, Message)
│   │   └── persistence.rs   # SQLite schema, migrations, CRUD operations
│   └── tui/
│       ├── mod.rs           # Terminal setup/teardown, run entrypoint
│       ├── app.rs           # Event loop, input handling, app state
│       ├── ui.rs            # Layout rendering (sidebar | chat+input | diff)
│       └── widgets/
│           ├── mod.rs
│           ├── sidebar.rs   # Session list with status indicators
│           ├── chat.rs      # Message display with role coloring
│           ├── input.rs     # Text input with cursor
│           ├── diff.rs      # Git diff viewer with syntax coloring
│           └── status_bar.rs # Mode indicator, status messages, agent count
└── examples/
    └── config.toml          # Example configuration file
```

---

## Database Schema

SQLite at `~/.local/share/kode/kode.db`, WAL mode, foreign keys enabled.

```sql
CREATE TABLE projects (
    id          TEXT PRIMARY KEY,   -- UUID v4
    name        TEXT NOT NULL,
    path        TEXT NOT NULL UNIQUE,
    created_at  TEXT NOT NULL       -- RFC 3339
);

CREATE TABLE sessions (
    id                TEXT PRIMARY KEY,
    project_id        TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name              TEXT NOT NULL,
    mode              TEXT NOT NULL DEFAULT 'agent',  -- 'plan' | 'agent'
    active            INTEGER NOT NULL DEFAULT 0,
    worktree_path     TEXT,
    claude_session_id TEXT,
    created_at        TEXT NOT NULL,
    updated_at        TEXT NOT NULL
);

CREATE TABLE messages (
    id          TEXT PRIMARY KEY,
    session_id  TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role        TEXT NOT NULL,      -- 'user' | 'assistant' | 'system'
    content     TEXT NOT NULL,
    timestamp   TEXT NOT NULL
);

CREATE INDEX idx_sessions_project ON sessions(project_id);
CREATE INDEX idx_messages_session ON messages(session_id);
```

Auto-migration on startup via `Database::migrate()`.

---

## XDG Directory Layout

| Path | Purpose |
|------|---------|
| `~/.config/kode/config.toml` | User configuration |
| `~/.local/share/kode/kode.db` | SQLite database |
| `~/.local/state/kode/log/kode.log` | Log file (tracing output) |

All paths resolved via the `dirs` crate. Created on first run if missing.

---

## Configuration

```toml
# ~/.config/kode/config.toml

# Path to the Claude CLI binary
claude_bin = "claude"

# Maximum concurrent Claude sessions
max_sessions = 5

# Default mode for new sessions: "plan" or "agent"
default_mode = "agent"

# Editor command for external file editing (defaults to $EDITOR or vim)
editor = "vim"
```

All fields optional with sensible defaults.

---

## CLI Interface

```
kode                    # Launch TUI (default)
kode init [path]        # Register current/given directory as a project
kode projects           # List all registered projects
kode sessions <project> # List sessions for a project
```

Implemented with `clap` derive macros.

---

## TUI Keybindings

### Normal Mode
| Key | Action |
|-----|--------|
| `q` | Quit |
| `?` | Show help |
| `j` / `↓` | Navigate down (sidebar or scroll chat) |
| `k` / `↑` | Navigate up |
| `Tab` | Cycle focus: sidebar → chat → input → diff → sidebar |
| `Shift+Tab` | Reverse cycle focus |
| `i` | Enter insert mode (focus input) |
| `:` | Enter command mode |
| `d` | Toggle diff panel |
| `n` | Create new session |
| `D` | Delete selected session |
| `Enter` | Select session in sidebar |
| `G` | Scroll chat to bottom |
| `g` | Scroll chat to top |
| `Ctrl+C` / `Ctrl+Q` | Force quit |

### Insert Mode
| Key | Action |
|-----|--------|
| `Esc` | Return to normal mode |
| `Enter` | Send message |
| Characters | Type into input buffer |
| `Backspace` | Delete character |

### Command Mode
| Command | Action |
|---------|--------|
| `:q` / `:quit` | Quit |
| `:new [name]` | Create named session |
| `:project <name>` | Switch active project |
| `:mode <plan\|agent>` | Set session mode |
| `:help` | Show commands |

---

## Implementation Phases

### Phase 1: Project Scaffold [Low complexity]

**Goal**: `cargo build` succeeds, XDG directories created.

- `cargo init` with all dependencies in `Cargo.toml`
- Create module file structure (empty `mod.rs` files compile)
- Implement `config.rs`: XDG path resolution, TOML loading with defaults
- Implement `cli.rs`: Clap derive structs
- Wire up `main.rs`: logging to file, config load, CLI dispatch
- **Checkpoint**: `cargo build` passes, `kode --help` works

### Phase 2: Git Worktree Manager [Medium complexity]

**Goal**: Programmatic worktree lifecycle management.

- `WorktreeManager::open(repo_path)` — verify git repo via `git2`
- `create(session_name)` — branch `kode/{name}`, worktree at `{repo}/.kode-worktrees/{name}/`
- `list()` — parse `git worktree list --porcelain`, filter `kode/` branches
- `remove(session_name)` — `git worktree remove --force` + `git branch -D`
- `prune()` — clean stale metadata
- **Decision**: Use `git2` for branch creation, shell out to `git` for worktree ops (git2's worktree API is limited)
- **Checkpoint**: Unit test creates worktree, lists it, removes it

### Phase 3: Claude CLI Integration [Medium complexity]

**Goal**: Spawn Claude in a PTY, stream output, forward input.

- `ClaudeSession::spawn(claude_bin, session_id, mode, working_dir, resume_id)`
- PTY via `portable-pty` (50×120 default size) — gives Claude proper terminal (colors, cursor)
- Reader thread: reads PTY → sends `ClaudeEvent::Output(String)` via `mpsc::unbounded_channel`
- Waiter thread: monitors process exit → sends `ClaudeEvent::Finished(exit_code)`
- `send_line(text)` — write to PTY master
- `interrupt()` — send `\x03` (Ctrl+C)
- `kill()` — drop PTY writer (SIGHUP)
- Mode support: `--allowedTools ""` for plan mode (read-only), no flag for agent
- Resume support: `--resume <session_id>` flag
- **Checkpoint**: Can start Claude, send "hello", receive streamed response

### Phase 4: State Management [Low complexity]

**Goal**: Persistent project/session/message storage.

- Domain types: `Project`, `Session`, `Message`, `MessageRole`
- `Database` struct wrapping `rusqlite::Connection`
- Auto-migration on open (CREATE TABLE IF NOT EXISTS)
- WAL mode + foreign keys enabled
- CRUD: insert/list/get/update/delete for all entities
- `AppState` facade over `Database` with convenience methods
- **Checkpoint**: Unit tests pass for all CRUD ops, cascade deletes work

### Phase 5: TUI Implementation [High complexity]

**Goal**: Full interactive terminal UI.

- Terminal setup/teardown: raw mode, alternate screen, mouse capture
- Layout: `Vertical[main_area, status_bar(1)]` → `Horizontal[sidebar(28), content]`
- Optional diff panel: three-column layout when toggled
- Sidebar widget: session list with `[P]`/`[A]` mode, `>` active indicator, color-coded
- Chat widget: messages with `you>` / `claude>` / `sys>` prefixes, line wrapping, scroll
- Input widget: text buffer with cursor position, insert/normal mode indicator
- Diff widget: `git diff` output with +/- coloring
- Status bar: mode badge (NORMAL/INSERT/COMMAND), message, focus indicator, agent count
- Event loop: 50ms poll tick, non-blocking agent polling, key dispatch
- Focus cycling: Tab/Shift+Tab across panes
- **Checkpoint**: Can navigate sidebar, select session, type message, see response

### Phase 6: Background Execution [Medium complexity]

**Goal**: Multiple concurrent Claude sessions.

- `HashMap<String, AgentHandle>` — session_id → (ClaudeSession, event_rx)
- `poll_agents()` called each frame — `try_recv()` drains all pending events
- Max session limit from config (`max_sessions`, default 5)
- Active session output appended to messages in real-time
- Background session output accumulated in `agent_output` buffer
- Session status synced to DB on start/finish
- Sidebar shows active indicator for running agents
- Status bar shows agent count
- **Checkpoint**: 3+ agents running in parallel, TUI stays responsive

### Phase 7: Audit [Medium complexity]

**Goal**: Production-quality code.

- [ ] Replace all `.unwrap()` in non-test code with proper error handling
- [ ] Verify all paths are XDG-compliant (no hardcoded paths)
- [ ] `cargo clippy -- -D warnings` passes clean
- [ ] `cargo test` passes (persistence tests, config tests)
- [ ] README with: build instructions, usage, keybindings, config reference
- [ ] Example `config.toml` in `examples/`
- [ ] License file (MIT)

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| PTY over raw process | Claude CLI expects a terminal for colors, interactive features, proper line buffering |
| SQLite over JSON files | Better concurrency, queryable, corruption-resistant, single-file backup |
| `tokio::mpsc` channels for output | Decouples CLI output rate from TUI render rate |
| Vim-like keybinds | Muscle memory for terminal users, modal editing is natural in TUI |
| `git2` + shell `git` hybrid | `git2` for branch operations, shell for worktree (git2's worktree API is incomplete) |
| Worktrees in `.kode-worktrees/` | Keeps worktrees colocated with repo, `.gitignore`-able, easy cleanup |
| Tracing to file, not stdout | stdout is owned by the TUI; logs go to `~/.local/state/kode/log/` |
| No custom error types | `anyhow` everywhere — this is a personal tool, not a library |

---

## Testing Strategy

- **Unit tests** in `state/persistence.rs`: CRUD operations using `tempfile` for in-memory DB
- **Unit tests** in `config.rs`: default loading, TOML parsing
- **Integration tests** in `git/worktree.rs`: create temp git repo, exercise worktree lifecycle
- **No TUI tests**: manual testing only (TUI testing frameworks add complexity for little value in a personal tool)

---

## What's Out of Scope (v1)

- Windows/macOS support
- Cloud sync, accounts, auth
- Voice input
- MCP server management (use Claude's native MCP support)
- Markdown rendering in chat (plain text with role prefixes is sufficient)
- File tree browser (use diff panel + external editor)
- Inline code editing (delegate to Claude + external editor)

---

## Potential v2 Features

| Feature | Complexity | Value |
|---------|-----------|-------|
| Markdown rendering (inline code blocks, bold) | Medium | High |
| File tree sidebar panel | Medium | Medium |
| Session search/filter | Low | Medium |
| Configurable keybindings | Medium | Low |
| Clipboard integration (copy messages) | Low | High |
| Session export (conversation → markdown file) | Low | Medium |
| Worktree auto-creation on session start | Low | High |
| Split pane for two sessions side-by-side | High | Medium |
