# Kode — Rust TUI for Claude Code Workflows

## Overview

A minimal, single-binary Rust TUI that replaces 1Code's Electron shell for personal Linux use. Manages Claude CLI sessions with git worktree isolation, background agent execution, and a vim-like terminal interface.

**Target**: Single Linux workstation (WSL2 or native). No cross-platform, no cloud, no accounts.

---

## Part 1: Agent Discovery

Before any code is written, the orchestrator must identify and define specialized agents. Each agent owns a vertical slice of the system and produces files that compile in isolation against shared type contracts.

### Agent Roster

| Agent ID | Name | Owns | Depends On |
|----------|------|------|------------|
| `A0` | **Scaffold** | `Cargo.toml`, `main.rs`, module stubs, directory layout | Nothing |
| `A1` | **Config** | `src/config.rs`, `examples/config.toml` | `A0` |
| `A2` | **CLI** | `src/cli.rs` | `A0` |
| `A3` | **Types** | `src/state/mod.rs` (domain types only: `Project`, `Session`, `Message`, enums) | `A0` |
| `A4` | **Persistence** | `src/state/persistence.rs` (SQLite schema, migrations, CRUD) | `A3` |
| `A5` | **Git** | `src/git/mod.rs`, `src/git/worktree.rs` | `A0` |
| `A6` | **Claude** | `src/claude/mod.rs`, `src/claude/session.rs` | `A3` |
| `A7` | **TUI Shell** | `src/tui/mod.rs`, `src/tui/app.rs`, `src/tui/ui.rs` | `A1`, `A3`, `A6` |
| `A8` | **Widget: Sidebar** | `src/tui/widgets/sidebar.rs` | `A7` |
| `A9` | **Widget: Chat** | `src/tui/widgets/chat.rs` | `A7` |
| `A10` | **Widget: Input** | `src/tui/widgets/input.rs` | `A7` |
| `A11` | **Widget: Diff** | `src/tui/widgets/diff.rs` | `A7` |
| `A12` | **Widget: StatusBar** | `src/tui/widgets/status_bar.rs` | `A7` |
| `A13` | **Integration** | `src/main.rs` (final wiring), `src/tui/widgets/mod.rs` | All above |
| `A14` | **Background Exec** | Extend `src/tui/app.rs` with agent pool + polling | `A6`, `A7` |
| `A15` | **Audit** | Clippy fixes, error handling sweep, README | `A13`, `A14` |

### Parallelism Map

```
Wave 0:  A0 (scaffold)
Wave 1:  A1, A2, A3, A5         ← all independent, only need A0
Wave 2:  A4, A6                  ← A4 needs A3; A6 needs A3
Wave 3:  A7                      ← needs A1, A3, A6
Wave 4:  A8, A9, A10, A11, A12  ← all independent, only need A7 types
Wave 5:  A13, A14                ← integration + background exec
Wave 6:  A15                     ← audit everything
```

---

## Part 2: Shared Contracts

Every agent receives these contracts before starting. They define the public interfaces that agents code against.

### 2.1 Directory Layout

```
kode/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs
│   ├── cli.rs
│   ├── config.rs
│   ├── git/
│   │   ├── mod.rs
│   │   └── worktree.rs
│   ├── claude/
│   │   ├── mod.rs
│   │   └── session.rs
│   ├── state/
│   │   ├── mod.rs
│   │   └── persistence.rs
│   └── tui/
│       ├── mod.rs
│       ├── app.rs
│       ├── ui.rs
│       └── widgets/
│           ├── mod.rs
│           ├── sidebar.rs
│           ├── chat.rs
│           ├── input.rs
│           ├── diff.rs
│           └── status_bar.rs
└── examples/
    └── config.toml
```

### 2.2 Crate Dependencies

```toml
[package]
name = "kode"
version = "0.1.0"
edition = "2024"

[dependencies]
ratatui = "0.29"
crossterm = "0.28"
tokio = { version = "1", features = ["full"] }
git2 = "0.19"
rusqlite = { version = "0.32", features = ["bundled"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
clap = { version = "4", features = ["derive"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dirs = "6"
portable-pty = "0.8"

[profile.release]
lto = true
strip = true
codegen-units = 1
```

### 2.3 Domain Types Contract (what `A3` produces, everyone consumes)

```rust
// src/state/mod.rs — public types

pub struct Project {
    pub id: String,           // UUID v4
    pub name: String,
    pub path: PathBuf,
    pub created_at: DateTime<Utc>,
}

pub struct Session {
    pub id: String,
    pub project_id: String,
    pub name: String,
    pub mode: SessionMode,    // from claude module
    pub active: bool,
    pub worktree_path: Option<PathBuf>,
    pub claude_session_id: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

pub struct Message {
    pub id: String,
    pub role: MessageRole,
    pub content: String,
    pub timestamp: DateTime<Utc>,
}

pub enum MessageRole { User, Assistant, System }

// Constructor helpers:
impl Message {
    pub fn user(content: String) -> Self;
    pub fn assistant(content: String) -> Self;
    pub fn system(content: String) -> Self;
}
```

### 2.4 Claude Types Contract (what `A6` produces)

```rust
// src/claude/session.rs — public types

pub enum SessionMode { Plan, Agent }
pub enum ClaudeEvent {
    Output(String),
    Started,
    Finished(Option<i32>),
    Error(String),
}

pub struct ClaudeSession { /* opaque */ }
impl ClaudeSession {
    pub fn spawn(claude_bin: &str, session_id: String, mode: SessionMode,
                 working_dir: &Path, resume: Option<&str>)
        -> Result<(Self, mpsc::UnboundedReceiver<ClaudeEvent>)>;
    pub fn send_input(&mut self, text: &str) -> Result<()>;
    pub fn send_line(&mut self, text: &str) -> Result<()>;
    pub fn interrupt(&mut self) -> Result<()>;
    pub fn kill(self);
}
```

### 2.5 Config Contract (what `A1` produces)

```rust
// src/config.rs — public types

pub struct Config {
    pub claude_bin: String,    // default: "claude"
    pub max_sessions: usize,   // default: 5
    pub default_mode: String,  // default: "agent"
    pub editor: String,        // default: $EDITOR or "vim"
}
impl Config {
    pub fn load() -> Result<Self>;
}
pub fn config_dir() -> Result<PathBuf>;  // ~/.config/kode/
pub fn data_dir() -> Result<PathBuf>;    // ~/.local/share/kode/
pub fn log_dir() -> Result<PathBuf>;     // ~/.local/state/kode/log/
pub fn db_path() -> Result<PathBuf>;     // ~/.local/share/kode/kode.db
```

### 2.6 TUI App State Contract (what `A7` produces, widgets consume)

```rust
// src/tui/app.rs — public fields widgets read

pub enum Focus { Sidebar, Chat, Input, Diff }
pub enum InputMode { Normal, Insert, Command }

pub struct App {
    pub focus: Focus,
    pub input_mode: InputMode,
    pub input_buffer: String,
    pub command_buffer: String,
    pub status_message: String,
    pub show_diff: bool,
    pub projects: Vec<Project>,
    pub sessions: Vec<Session>,
    pub messages: Vec<Message>,
    pub chat_scroll: u16,
    pub sidebar_index: usize,
    pub active_session_id: Option<String>,
    pub active_project_id: Option<String>,
    // agents field is pub(crate) for status_bar agent count
}
```

### 2.7 Database Schema Contract

```sql
CREATE TABLE projects (
    id TEXT PRIMARY KEY, name TEXT NOT NULL,
    path TEXT NOT NULL UNIQUE, created_at TEXT NOT NULL
);
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    project_id TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL, mode TEXT NOT NULL DEFAULT 'agent',
    active INTEGER NOT NULL DEFAULT 0, worktree_path TEXT,
    claude_session_id TEXT, created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role TEXT NOT NULL, content TEXT NOT NULL, timestamp TEXT NOT NULL
);
CREATE INDEX idx_sessions_project ON sessions(project_id);
CREATE INDEX idx_messages_session ON messages(session_id);
```

---

## Part 3: Task Breakdown

Each task is a self-contained unit of work for one agent. Tasks specify:
- **Agent**: who does it
- **Inputs**: what files/contracts must exist before starting
- **Outputs**: exact files produced
- **Acceptance**: how the orchestrator verifies success
- **Instructions**: what to implement

### Wave 0 — Foundation

#### T0.1: Initialize Cargo project
- **Agent**: `A0`
- **Inputs**: None
- **Outputs**: `kode/Cargo.toml`, `kode/src/main.rs`
- **Acceptance**: `cargo check` passes (main.rs can be a placeholder `fn main() {}`)
- **Instructions**:
  1. Run `cargo init kode --name kode` in a new directory (NOT inside the 1code repo)
  2. Replace `Cargo.toml` contents with the exact crate dependencies from §2.2
  3. Write `src/main.rs` as:
     ```rust
     mod cli;
     mod config;
     mod claude;
     mod git;
     mod state;
     mod tui;
     fn main() { println!("kode stub"); }
     ```

#### T0.2: Create module stubs
- **Agent**: `A0`
- **Inputs**: T0.1
- **Outputs**: All `mod.rs` files with placeholder contents
- **Acceptance**: `cargo check` passes with no missing module errors
- **Instructions**:
  1. Create directory structure from §2.1
  2. Each `mod.rs` file declares its submodules:
     - `src/git/mod.rs` → `pub mod worktree;`
     - `src/claude/mod.rs` → `pub mod session;`
     - `src/state/mod.rs` → `pub mod persistence;`
     - `src/tui/mod.rs` → `pub mod app; pub mod ui; pub mod widgets;`
     - `src/tui/widgets/mod.rs` → `pub mod sidebar; pub mod chat; pub mod input; pub mod diff; pub mod status_bar;`
  3. Each leaf `.rs` file gets an empty body (or a single `// TODO` comment)
  4. `src/cli.rs`, `src/config.rs` get empty bodies

---

### Wave 1 — Independent Modules (run in parallel)

#### T1.1: Config module
- **Agent**: `A1`
- **Inputs**: T0.2 (module stubs exist)
- **Outputs**: `src/config.rs`, `examples/config.toml`
- **Acceptance**: `cargo check` passes; unit test for `Config::default()` and `Config::load()` with missing file
- **Instructions**:
  1. Define `Config` struct with `serde::Deserialize + Serialize + Debug + Clone`
  2. Fields: `claude_bin` (String, default "claude"), `max_sessions` (usize, default 5), `default_mode` (String, default "agent"), `editor` (String, default `$EDITOR` or "vim")
  3. Implement `Config::load()`: read `config_dir()/config.toml`, parse with `toml::from_str`, fall back to `Config::default()` if file missing
  4. Implement `Default` for `Config`
  5. Public functions: `config_dir()`, `data_dir()`, `log_dir()`, `db_path()` — all return `Result<PathBuf>` using `dirs` crate
  6. `log_dir` uses `dirs::state_dir()` with fallback to `dirs::data_dir()`
  7. Write `examples/config.toml` with all fields documented

#### T1.2: CLI module
- **Agent**: `A2`
- **Inputs**: T0.2
- **Outputs**: `src/cli.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. Use `clap::Parser` derive on `Cli` struct
  2. Add `#[command(version, about)]` for auto version/help
  3. `command: Option<Command>` field with `#[command(subcommand)]`
  4. Define `Command` enum:
     - `Init { path: Option<PathBuf> }` — "Initialize a project in the given (or current) directory"
     - `Projects` — "List all registered projects"
     - `Sessions { project: String }` — "List sessions for a project"
  5. When `command` is `None`, the TUI launches (handled in `main.rs`)

#### T1.3: Domain types
- **Agent**: `A3`
- **Inputs**: T0.2
- **Outputs**: `src/state/mod.rs` (types only, no persistence)
- **Acceptance**: `cargo check` passes; types are importable from `crate::state`
- **Instructions**:
  1. Declare `pub mod persistence;` at top
  2. Define `Project`, `Session`, `Message`, `MessageRole` exactly as in §2.3
  3. All types derive `Debug, Clone, Serialize, Deserialize`
  4. `MessageRole` has `Display` impl (returns "user"/"assistant"/"system") and `from_str` class method
  5. `Message` has `user()`, `assistant()`, `system()` constructors using `Uuid::new_v4()` and `Utc::now()`
  6. `SessionMode` is referenced from `crate::claude::SessionMode` — for now, define a local placeholder enum `SessionMode { Plan, Agent }` with `Display`, `from_str`, and `Serialize/Deserialize`. The Claude agent will later provide the real one; the integration agent will reconcile.
  7. Define `AppState` struct holding a `persistence::Database` with these public methods:
     - `new(config: &Config) -> Result<Self>`
     - `add_project(name: String, path: PathBuf) -> Result<Project>`
     - `list_projects() -> Result<Vec<Project>>`
     - `get_project_by_name(name: &str) -> Result<Option<Project>>`
     - `create_session(project_id: &str, name: String, mode: SessionMode) -> Result<Session>`
     - `list_sessions_for_project(project_name: &str) -> Result<Vec<Session>>`
     - `list_all_sessions() -> Result<Vec<Session>>`
     - `get_session(session_id: &str) -> Result<Option<Session>>`
     - `update_session_active(session_id: &str, active: bool) -> Result<()>`
     - `add_message(session_id: &str, message: Message) -> Result<()>`
     - `get_messages(session_id: &str) -> Result<Vec<Message>>`
     - `delete_session(session_id: &str) -> Result<()>`

#### T1.4: Git worktree manager
- **Agent**: `A5`
- **Inputs**: T0.2
- **Outputs**: `src/git/mod.rs`, `src/git/worktree.rs`
- **Acceptance**: `cargo check` passes; `#[cfg(test)]` tests pass for create/list/remove (using temp git repo)
- **Instructions**:
  1. `src/git/mod.rs`: `pub mod worktree; pub use worktree::WorktreeManager;`
  2. `WorktreeManager` struct holds `repo_path: PathBuf`
  3. `open(repo_path: &Path) -> Result<Self>`: verify path is a git repo via `git2::Repository::open`
  4. `create(session_name: &str) -> Result<WorktreeInfo>`:
     - Open repo with `git2`, get HEAD commit
     - Create branch `kode/{session_name}` via `repo.branch()`
     - Worktree path = `{repo_path}/.kode-worktrees/{session_name}/`
     - Shell out: `git worktree add --checkout {worktree_path} {branch}` (git2's worktree API is limited)
     - Return `WorktreeInfo { name, path, branch }`
  5. `list() -> Result<Vec<WorktreeInfo>>`:
     - Shell out: `git worktree list --porcelain`
     - Parse output: `worktree <path>` and `branch refs/heads/<name>` lines
     - Filter to branches starting with `kode/` and paths under `.kode-worktrees/`
  6. `remove(session_name: &str) -> Result<()>`:
     - Shell out: `git worktree remove --force {path}`
     - Shell out: `git branch -D kode/{session_name}`
     - Warn (don't fail) if already gone
  7. `prune() -> Result<()>`: shell out `git worktree prune`
  8. `WorktreeInfo` struct: `name: String, path: PathBuf, branch: String` — derive `Debug, Clone`
  9. Private helper `worktrees_root(&self) -> PathBuf` returns `self.repo_path.join(".kode-worktrees")`
  10. Tests: create a temp dir, `git init`, create worktree, list it, remove it, verify gone

---

### Wave 2 — Depends on Types (run in parallel)

#### T2.1: SQLite persistence
- **Agent**: `A4`
- **Inputs**: T1.3 (domain types)
- **Outputs**: `src/state/persistence.rs`
- **Acceptance**: `cargo check` passes; `cargo test` passes for all CRUD + cascade delete
- **Instructions**:
  1. `Database` struct wrapping `rusqlite::Connection`
  2. `Database::open(config: &Config) -> Result<Self>`:
     - `std::fs::create_dir_all(data_dir()?)`
     - `Connection::open(db_path()?)`
     - `PRAGMA journal_mode=WAL`
     - `PRAGMA foreign_keys=ON`
     - Call `self.migrate()`
  3. `migrate(&self)`: execute batch SQL from §2.7 (all `CREATE TABLE IF NOT EXISTS`)
  4. Implement these methods matching the `AppState` facade in T1.3:
     - `insert_project(&mut self, name: &str, path: &Path) -> Result<Project>` — generate UUID, use `Utc::now().to_rfc3339()` for timestamps
     - `list_projects(&self) -> Result<Vec<Project>>` — ORDER BY name
     - `get_project_by_name(&self, name: &str) -> Result<Option<Project>>`
     - `insert_session(&mut self, project_id: &str, name: &str, mode: SessionMode) -> Result<Session>`
     - `list_sessions(&self, project_id: &str) -> Result<Vec<Session>>` — ORDER BY updated_at DESC
     - `list_all_sessions(&self) -> Result<Vec<Session>>`
     - `get_session(&self, session_id: &str) -> Result<Option<Session>>`
     - `update_session_active(&mut self, session_id: &str, active: bool) -> Result<()>` — also update `updated_at`
     - `insert_message(&mut self, session_id: &str, message: &Message) -> Result<()>`
     - `get_messages(&self, session_id: &str) -> Result<Vec<Message>>` — ORDER BY timestamp ASC
     - `delete_session(&mut self, session_id: &str) -> Result<()>` — messages cascade via FK
  5. Parse DateTime from stored RFC 3339 strings, fall back to `Utc::now()` on parse failure
  6. Tests (use `tempfile::TempDir` for test DB, bypass `Config`/XDG):
     - `test_project_crud`: insert, list, get by name
     - `test_session_crud`: insert, update active, get, list
     - `test_message_crud`: insert, get ordered
     - `test_cascade_delete`: delete session, verify messages gone

#### T2.2: Claude CLI session
- **Agent**: `A6`
- **Inputs**: T1.3 (for `SessionMode` type)
- **Outputs**: `src/claude/mod.rs`, `src/claude/session.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. `src/claude/mod.rs`: `pub mod session; pub use session::{ClaudeSession, ClaudeEvent, SessionMode};`
  2. Define `SessionMode` enum: `Plan, Agent` with `Display` (returns "plan"/"agent"), `from_str`, `Serialize`, `Deserialize`, `Debug, Clone, Copy, PartialEq, Eq`
  3. Define `ClaudeEvent` enum per §2.4 — derive `Debug, Clone`
  4. Define `ClaudeSession` struct:
     - Fields: `session_id: String`, `mode: SessionMode`, `working_dir: PathBuf`, `writer: Option<Box<dyn Write + Send>>`, `kill_tx: Option<oneshot::Sender<()>>`
     - `session_id` and `mode` are `pub`, rest private
  5. `spawn()` implementation:
     - Open PTY via `portable_pty::native_pty_system()`, size 50x120
     - Build `CommandBuilder` with `claude_bin` executable
     - For `Plan` mode: add args `--allowedTools ""`
     - For resume: add args `--resume {session_id}`
     - Set cwd to `working_dir`
     - Get master writer and reader from PTY
     - Spawn child via `pty_pair.slave.spawn_command(cmd)`
     - Drop slave after spawn
     - Create `mpsc::unbounded_channel::<ClaudeEvent>()`
     - Spawn **std::thread** (not tokio) for reader: loop read 4096 bytes, send `Output(String)`, break on EOF or error
     - Spawn **std::thread** for waiter: `child.wait()`, send `Finished(exit_code)`
     - Send `Started` event immediately
     - Return `(Self, receiver)`
  6. `send_input(&mut self, text: &str)`: write bytes to PTY writer, flush
  7. `send_line(&mut self, text: &str)`: call `send_input` with `\n` appended
  8. `interrupt(&mut self)`: `send_input("\x03")`
  9. `kill(mut self)`: send kill signal via oneshot, drop writer
  10. `Drop` impl: send kill signal if still held
  11. Add `use std::io::Read;` at module level for reader thread

---

### Wave 3 — TUI Core

#### T3.1: TUI shell (terminal setup + app struct + layout)
- **Agent**: `A7`
- **Inputs**: T1.1 (Config), T1.3 (types), T2.2 (ClaudeSession, ClaudeEvent)
- **Outputs**: `src/tui/mod.rs`, `src/tui/app.rs`, `src/tui/ui.rs`
- **Acceptance**: `cargo check` passes
- **Instructions for `src/tui/mod.rs`**:
  1. Declare `pub mod app; pub mod ui; pub mod widgets;`
  2. Public function `pub async fn run(app_state: AppState, config: Config) -> Result<()>`:
     - `enable_raw_mode()`
     - `execute!(stdout, EnterAlternateScreen, EnableMouseCapture)`
     - Create `CrosstermBackend`, `Terminal::new(backend)`, `terminal.clear()`
     - Call `App::new(app_state, config).run(&mut terminal).await`
     - On exit: `disable_raw_mode()`, `LeaveAlternateScreen`, `DisableMouseCapture`, `show_cursor()`

- **Instructions for `src/tui/app.rs`**:
  1. Define `Focus` enum: `Sidebar, Chat, Input, Diff` — derive `Debug, Clone, Copy, PartialEq, Eq`
  2. Define `InputMode` enum: `Normal, Insert, Command` — same derives
  3. Private struct `AgentHandle { session: ClaudeSession, event_rx: mpsc::UnboundedReceiver<ClaudeEvent> }`
  4. Public struct `App` with all fields from §2.6, plus:
     - `pub state: AppState`
     - `pub config: Config`
     - `pub should_quit: bool`
     - `pub(crate) agents: HashMap<String, AgentHandle>`
     - `pub agent_output: HashMap<String, Vec<String>>`
  5. `App::new(state, config)`: load `projects` and `sessions` from state, set defaults (focus=Sidebar, input_mode=Normal, status_message="Press ? for help")
  6. `App::run(&mut self, terminal) -> Result<()>`: loop that:
     - Calls `terminal.draw(|frame| ui::render(frame, self))`
     - Calls `self.poll_agents()` (empty method for now — A14 fills it)
     - Polls `crossterm::event::poll(Duration::from_millis(50))`
     - On `Event::Key(key)`: calls `self.handle_key(key)`
     - Breaks on `self.should_quit`
  7. Input handling — three methods dispatched by `input_mode`:
     - `handle_normal_key`: implement all keybinds from Normal Mode table (§ TUI Keybindings below)
     - `handle_insert_key`: Esc exits insert, Enter sends message, Char appends, Backspace pops
     - `handle_command_key`: Esc cancels, Enter executes command, Char/Backspace edit buffer
  8. Command execution (`execute_command`): parse `:q`, `:new [name]`, `:project <name>`, `:mode <plan|agent>`, `:help`
  9. Navigation helpers: `navigate_down`, `navigate_up`, `cycle_focus`, `cycle_focus_reverse`
  10. Session management: `select_current`, `create_new_session`, `create_named_session`, `delete_selected_session`, `switch_project`
  11. Message sending: `send_message` stores user message in DB, forwards to active agent or starts new one
  12. Claude session starting: `start_claude_session` checks max_sessions, determines working dir, calls `ClaudeSession::spawn`, stores handle
  13. Agent polling (stub): `poll_agents` — empty method body, placeholder for A14

- **Instructions for `src/tui/ui.rs`**:
  1. `pub fn render(frame: &mut Frame, app: &App)`:
     - Split area vertically: `[Min(3), Length(1)]` → main + status bar
     - If `app.show_diff`: split main horizontally into 3 columns `[Length(28), Percentage(50), Min(30)]`
     - Else: split into 2 columns `[Length(28), Min(40)]`
     - Split content column vertically: `[Min(5), Length(3)]` → chat + input
     - Call widget render functions: `sidebar::render`, `chat::render`, `input::render`, `status_bar::render`
     - Conditionally call `diff::render` when `show_diff` is true

---

### Wave 4 — Widgets (all run in parallel)

Each widget is a standalone rendering function: `pub fn render(frame: &mut Frame, app: &App, area: Rect)`.

#### T4.1: Sidebar widget
- **Agent**: `A8`
- **Inputs**: T3.1 (App struct, Focus enum)
- **Outputs**: `src/tui/widgets/sidebar.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. Build `Vec<ListItem>` from `app.sessions`
  2. Each item shows: `{status} [{mode}] {name}` where status is `>` if active, ` ` otherwise; mode is `P` or `A`
  3. Style: active session = Yellow bold, focused+selected = White bold, running = Green, default = Gray
  4. Title: `" Sessions ({project_name}) "` if project selected, else `" Sessions "`
  5. Border: Cyan when focused (`app.focus == Focus::Sidebar`), DarkGray otherwise
  6. Use `ListState` with `app.sidebar_index` as selection
  7. Render with `frame.render_stateful_widget`

#### T4.2: Chat widget
- **Agent**: `A9`
- **Inputs**: T3.1 (App struct)
- **Outputs**: `src/tui/widgets/chat.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. If `app.messages` is empty: show centered paragraph "No messages yet. Press 'i' to start typing." in DarkGray
  2. Otherwise, iterate messages building `Vec<Line>`:
     - User: prefix `"you> "` in Cyan
     - Assistant: prefix `"claude> "` in Green
     - System: prefix `"sys> "` in Yellow
  3. Multi-line content: first line gets prefix, subsequent lines get whitespace padding to align
  4. Blank line between messages
  5. Wrap with `Wrap { trim: false }`, scroll with `app.chat_scroll`
  6. Border: Cyan when focused, DarkGray otherwise; title `" Chat "`

#### T4.3: Input widget
- **Agent**: `A10`
- **Inputs**: T3.1 (App struct, InputMode)
- **Outputs**: `src/tui/widgets/input.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. Render `app.input_buffer` as a `Paragraph`
  2. Border color: Yellow in insert mode + focused, Cyan if just focused, DarkGray otherwise
  3. Title: `" Input (Esc to exit) "` in insert mode, `" Input (i to type) "` otherwise
  4. When in insert mode: call `frame.set_cursor_position()` at end of input text (x = area.x + buffer.len() + 1, y = area.y + 1; clamp x to area.right() - 2)

#### T4.4: Diff widget
- **Agent**: `A11`
- **Inputs**: T3.1 (App struct)
- **Outputs**: `src/tui/widgets/diff.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. Determine working directory: prefer `session.worktree_path`, fall back to project path, then "No worktree or project path"
  2. Shell out: `git diff --stat --patch` in that directory
  3. Color each line: `+` lines Green (not `+++`), `-` lines Red (not `---`), `@@` lines Cyan, `diff`/`index` lines Yellow, rest default
  4. Render as `Paragraph` with `Wrap { trim: false }`
  5. Border: Cyan when focused, DarkGray otherwise; title `" Diff "`

#### T4.5: Status bar widget
- **Agent**: `A12`
- **Inputs**: T3.1 (App struct, InputMode, Focus)
- **Outputs**: `src/tui/widgets/status_bar.rs`
- **Acceptance**: `cargo check` passes
- **Instructions**:
  1. Mode badge: `" NORMAL "` (Blue bg, White), `" INSERT "` (Green bg, Black), `" COMMAND "` (Yellow bg, Black) — all bold
  2. Status message: `app.status_message` in White
  3. Right side: focus name + agent count (e.g., `" chat  agents:3 "`) in DarkGray
  4. Pad middle with spaces to right-align the right side
  5. Render as single-line `Paragraph` with Black background

---

### Wave 5 — Integration (run in parallel)

#### T5.1: Wire main.rs
- **Agent**: `A13`
- **Inputs**: All Wave 0–4 outputs
- **Outputs**: `src/main.rs`, `src/tui/widgets/mod.rs`
- **Acceptance**: `cargo build` succeeds (full compile, not just check)
- **Instructions**:
  1. `src/tui/widgets/mod.rs` declares all widget submodules
  2. `src/main.rs`:
     - Declare all top-level modules: `mod cli; mod config; mod claude; mod git; mod state; mod tui;`
     - `#[tokio::main] async fn main() -> Result<()>`
     - Parse CLI args with `Cli::parse()`
     - Initialize tracing: create `log_dir()`, open file, configure `tracing_subscriber::fmt()` with `env-filter`, `with_writer(file)`, `with_ansi(false)`
     - Load config with `Config::load()`
     - Match on `cli.command`:
       - `Init { path }`: canonicalize path (default to cwd), call `AppState::add_project`, print result
       - `Projects`: list and print projects
       - `Sessions { project }`: list and print sessions for project
       - `None`: call `tui::run(app_state, config).await`
  3. Resolve any type conflicts between `claude::SessionMode` and `state::SessionMode`:
     - The canonical `SessionMode` lives in `claude::session`
     - `state/mod.rs` should re-export or use `crate::claude::SessionMode`
     - Remove duplicate definition if A3 created a placeholder

#### T5.2: Background agent execution
- **Agent**: `A14`
- **Inputs**: T3.1 (app.rs), T2.2 (ClaudeSession)
- **Outputs**: Modified `src/tui/app.rs` (extend `poll_agents` method)
- **Acceptance**: `cargo check` passes; `poll_agents` is fully implemented
- **Instructions**:
  1. Replace the stub `poll_agents` with full implementation:
     - Collect all session_ids from `self.agents` keys
     - For each agent, `try_recv()` in a loop to drain all pending events
     - On `ClaudeEvent::Output(text)`:
       - Append to `self.agent_output` buffer
       - If session is active (`active_session_id`), also create `Message::assistant(text)` and persist + append to `self.messages`
     - On `ClaudeEvent::Finished(code)`:
       - Track session id for removal
       - Call `self.state.update_session_active(id, false)`
     - On `ClaudeEvent::Error(e)`: set `self.status_message`
     - After loop: remove finished agents from HashMap
     - If any agents were polled: refresh `self.sessions` from DB

---

### Wave 6 — Audit

#### T6.1: Error handling sweep
- **Agent**: `A15`
- **Inputs**: Full compiled codebase
- **Outputs**: Modified source files
- **Acceptance**: `grep -r '\.unwrap()' src/ | grep -v '#\[cfg(test)\]'` returns no non-test unwraps (allow `unwrap_or`, `unwrap_or_else`, `unwrap_or_default`)
- **Instructions**:
  1. Search all `.rs` files for `.unwrap()` outside `#[cfg(test)]` blocks
  2. Replace with `.context("description")?`, `.unwrap_or(default)`, or `.unwrap_or_else(|| ...)` as appropriate
  3. Exception: `expect()` is acceptable in truly unreachable paths (e.g., regex compilation of static strings)

#### T6.2: Clippy pass
- **Agent**: `A15`
- **Inputs**: T6.1
- **Outputs**: Modified source files
- **Acceptance**: `cargo clippy -- -D warnings` exits 0
- **Instructions**:
  1. Run `cargo clippy -- -D warnings`
  2. Fix every warning. Common fixes:
     - Replace `&String` params with `&str`
     - Replace `&Vec<T>` params with `&[T]`
     - Use `if let` instead of single-arm `match`
     - Remove unused imports
     - Fix needless borrows

#### T6.3: Test pass
- **Agent**: `A15`
- **Inputs**: T6.2
- **Outputs**: Possibly modified test code
- **Acceptance**: `cargo test` exits 0
- **Instructions**:
  1. Run `cargo test`
  2. Fix any failing tests
  3. Ensure persistence tests use `tempfile` (add `tempfile` to `[dev-dependencies]` if missing)

#### T6.4: README
- **Agent**: `A15`
- **Inputs**: Full working codebase
- **Outputs**: `README.md`
- **Acceptance**: File exists, contains build/run/keybind sections
- **Instructions**:
  Write README with these sections:
  1. **kode** — one-line description
  2. **Build**: `cargo build --release`, binary at `target/release/kode`
  3. **Prerequisites**: Rust toolchain, Claude CLI installed
  4. **Usage**: `kode` (TUI), `kode init`, `kode projects`, `kode sessions <name>`
  5. **Configuration**: location, all fields with defaults
  6. **Keybindings**: Normal, Insert, Command mode tables
  7. **Architecture**: brief module overview
  8. **License**: MIT

---

## Part 4: Orchestrator Protocol

The orchestrator executes the plan using this protocol:

### Step 1: Validate Environment
```
- Verify: rustc, cargo, git available
- Verify: target directory for new repo exists and is empty
```

### Step 2: Execute Waves
```
For each wave:
  1. Dispatch all tasks in the wave to agents (in parallel where marked)
  2. Collect outputs from each agent
  3. Verify acceptance criteria for each task:
     - Run the specified cargo/test/grep command
     - If FAIL: send error output back to the same agent for retry (max 3 retries)
  4. Only proceed to next wave when ALL tasks in current wave pass
```

### Step 3: Reconciliation Points
```
After Wave 1: Ensure A3's SessionMode placeholder matches A6's plan
After Wave 4: Ensure all widget render() signatures match what ui.rs calls
After Wave 5: Full `cargo build` — this is the first time ALL code compiles together
```

### Step 4: Final Gate
```
Run in sequence:
  cargo build
  cargo test
  cargo clippy -- -D warnings
ALL must exit 0. If any fail, return to A15 for fixes.
```

---

## Part 5: Reference

### TUI Keybindings

#### Normal Mode
| Key | Action |
|-----|--------|
| `q` | Quit |
| `?` | Show help in status bar |
| `j` / `↓` | Navigate down |
| `k` / `↑` | Navigate up |
| `Tab` | Cycle focus forward |
| `Shift+Tab` | Cycle focus backward |
| `i` | Enter insert mode |
| `:` | Enter command mode |
| `d` | Toggle diff panel |
| `n` | Create new session |
| `D` | Delete selected session |
| `Enter` | Select current item |
| `G` | Scroll to bottom |
| `g` | Scroll to top |
| `Ctrl+C` / `Ctrl+Q` | Force quit |

#### Insert Mode
| Key | Action |
|-----|--------|
| `Esc` | Return to normal mode |
| `Enter` | Send message |
| Characters | Append to input |
| `Backspace` | Delete last character |

#### Command Mode
| Command | Action |
|---------|--------|
| `:q` / `:quit` | Quit |
| `:new [name]` | Create named session |
| `:project <name>` | Switch project |
| `:mode <plan\|agent>` | Set session mode |
| `:help` | Show available commands |

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| PTY over raw process | Claude CLI needs a terminal for colors and interactive features |
| SQLite over JSON files | Concurrency-safe, queryable, corruption-resistant |
| `tokio::mpsc` channels | Decouples CLI output rate from TUI render rate |
| `git2` + shell `git` hybrid | git2 for branches, shell for worktree (git2 API gap) |
| Worktrees in `.kode-worktrees/` | Colocated, gitignore-able, easy cleanup |
| Tracing to file only | stdout is owned by the TUI |
| `anyhow` everywhere | Personal tool, not a library — no custom error types needed |
| `std::thread` for PTY I/O | PTY read is blocking I/O; tokio tasks would need `spawn_blocking` anyway |

### Out of Scope (v1)

- Windows/macOS
- Cloud sync, accounts, auth
- Voice input
- MCP server management
- Markdown rendering
- File tree browser
- Inline code editing
