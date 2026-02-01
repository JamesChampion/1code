# Testing Patterns

**Analysis Date:** 2026-02-01

## Test Framework

**Status:** Not configured

**Note:** `package.json` explicitly states `"There is no test suite or linter configured in this project."` per `CLAUDE.md`. No test files found (searched for `*.test.ts`, `*.spec.ts`).

**Testing Approach:**
- Manual testing and code review
- Integration testing via actual Electron runtime (`bun run dev`)
- End-to-end testing via live Claude API interactions

## Available Test Tools (Installed but Unconfigured)

**Runner:**
- Jest (potentially, via `@tanstack/react-query` ecosystem)
- Vitest (not detected in config)

**Assertion Library:**
- None explicitly configured

**Test Directories:**
- No `__tests__`, `tests`, or `test` directories found in source tree
- Build output (`out/`) contains compiled code only

## Run Commands

Currently available from `package.json`:

```bash
bun run ts:check              # TypeScript validation (uses tsgo)
bun run dev                   # Start Electron with hot reload (functional testing)
bun run build                 # Compile TypeScript
```

No dedicated test commands exist. No coverage reporting configured.

## Manual Testing Strategy

**TypeScript Validation:**
- `bun run ts:check` runs tsgo for type checking
- Catches type errors before runtime

**Development Testing:**
- `bun run dev` launches Electron with hot module reload
- Developers manually test features in the running app
- Real-time feedback for UI/backend integration

**Incremental Verification:**
- Compile step catches syntax errors
- tRPC type safety validates IPC contracts
- React component prop types catch misuse

## Code Organization for Testing (No Test Files)

**Separation of Concerns (Enables Future Testing):**

**Main Process (testable if tests were added):**
- Git operations isolated: `src/main/lib/git/`
- Database operations: `src/main/lib/db/`
- Router definitions: `src/main/lib/trpc/routers/`
- Auth logic: `src/main/auth-manager.ts`
- Each module is self-contained with clear inputs/outputs

**Renderer (testable if tests were added):**
- UI components: `src/renderer/features/agents/components/`
- Stores: `src/renderer/lib/stores/`
- Hooks: `src/renderer/lib/hooks/`
- Utilities: `src/renderer/lib/utils/`
- Each component/hook accepts props and returns values deterministically

## Data Validation (Functional Tests)

**Input Validation - Zod Schemas:**

All tRPC procedures use Zod for runtime validation. Example from `projects.ts`:

```typescript
create: publicProcedure
  .input(z.object({
    path: z.string(),
    name: z.string().optional()
  }))
  .mutation(async ({ input }) => { ... })
```

Validation happens automatically before handler execution. Invalid input rejected with typed error.

**Error Boundaries:**

Git operations use custom error types: `PathValidationError` in `src/main/lib/git/security/path-validation.ts`

```typescript
throw new PathValidationError(
  "Path escapes worktree",
  "SYMLINK_ESCAPE"
)
```

Enables error classification without catch blocks.

## Integration Points (Where Tests Would Be Critical)

**Git Operations:**
- `src/main/lib/git/worktree.ts` - Worktree creation/deletion (critical, high impact)
- `src/main/lib/git/git-operations.ts` - Commit, push, branch operations
- `src/main/lib/git/status.ts` - File status detection (caching layer)
- These are integration-tested by live Electron runtime

**Database Operations:**
- `src/main/lib/db/` - SQLite initialization, migrations, queries
- Schema-driven (Drizzle ORM) catches most type errors
- Auto-migration on startup tested via actual app startup

**tRPC Routers:**
- `src/main/lib/trpc/routers/` - All 17 routers
- Type-safe due to Zod + TypeScript
- Integration tested via Electron renderer calling tRPC procedures

**Authentication:**
- `src/main/auth-manager.ts` - OAuth token refresh, storage
- `src/main/lib/oauth.ts` - OAuth flows
- Tested via actual login flows in live app

## Example: How Code Is Structured for Testability (If Tests Existed)

**Git Factory (Could Be Unit Tested):**

Location: `src/main/lib/git/git-factory.ts`

```typescript
/**
 * @param worktreePath - Path to the git worktree/repository
 * @param timeout - Timeout in milliseconds (defaults to LOCAL timeout)
 * @returns Configured SimpleGit instance
 */
export function createSimpleGitInstance(
  worktreePath: string,
  timeout: number = LOCAL
): SimpleGit { ... }

/**
 * @param worktreePath - Path to the git worktree
 * @param operation - Async function to execute
 * @returns Result of the operation
 */
export async function executeGitOperation<T>(
  worktreePath: string,
  operation: (git: SimpleGit) => Promise<T>
): Promise<T> { ... }
```

Pure functions with clear contracts - testable with mock SimpleGit instances.

**Store Mutations (Could Be Unit Tested):**

Location: `src/renderer/lib/stores/sub-chat-store.ts`

```typescript
export const useAgentSubChatStore = create<AgentSubChatStore>((set, get) => ({
  activeSubChatId: null,
  openSubChatIds: [],

  setActiveSubChat: (subChatId) => {
    const { chatId } = get()
    set({ activeSubChatId: subChatId })
    if (chatId) saveToLS(chatId, "active", subChatId)
  },

  addToOpenSubChats: (subChatId) => {
    const { openSubChatIds, chatId } = get()
    if (openSubChatIds.includes(subChatId)) return
    const newIds = [...openSubChatIds, subChatId]
    set({ openSubChatIds: newIds })
    if (chatId) saveToLS(chatId, "open", newIds)
  },
}))
```

State transitions deterministic - could test with mock localStorage and verify state mutations.

**React Component Props (Type-Checked):**

Location: `src/renderer/features/agents/components/agent-send-button.tsx`

```typescript
interface AgentSendButtonProps {
  isStreaming?: boolean
  isSubmitting?: boolean
  disabled?: boolean
  onClick: () => void
  onStop?: () => void
  isPlanMode?: boolean
  hasContent?: boolean
  showVoiceInput?: boolean
  isRecording?: boolean
  isTranscribing?: boolean
}

export function AgentSendButton({
  isStreaming = false,
  isSubmitting = false,
  disabled = false,
  onClick,
  onStop,
  // ...
}: AgentSendButtonProps) {
  // All prop combinations tested implicitly by type system
  const shouldShowQueueArrow = isStreaming && hasContent
  const isVoiceMode = showVoiceInput && !isStreaming && !hasContent
  // ...
}
```

TypeScript ensures all prop combinations are valid. Renders deterministically based on prop state.

## Testing Philosophy (Current)

- **Type Safety > Test Coverage**: TypeScript strict mode catches many errors
- **Integration > Unit**: Features tested end-to-end in live Electron
- **Code Review > Automation**: Manual review catches logic errors
- **Incremental Feedback**: Hot reload in dev mode provides immediate feedback

## Coverage

**Status:** Not measured

No coverage tools configured. Given the codebase size (~1200 source files), adding test coverage would require:
- Jest or Vitest configuration
- Test runner setup
- Mock implementations for Electron APIs, git operations, database
- Estimated 1000+ tests for 80% coverage across three processes (main, preload, renderer)

## Future Testing Setup (If Needed)

**Recommended Approach:**
1. Vitest for unit tests (fast, modern, TypeScript-first)
2. @testing-library/react for component tests
3. Playwright for E2E tests (Electron-compatible)
4. Mock electron, better-sqlite3, simple-git in unit tests
5. Start with critical paths: git operations, auth, database migrations

**Entry Points for Tests:**
- `src/main/lib/git/` - Git operations (high value, isolated)
- `src/main/lib/db/` - Database operations (schema-driven, migrations)
- `src/renderer/lib/stores/` - State management (pure Zustand mutations)
- `src/renderer/features/agents/components/` - UI rendering (prop-driven)

---

*Testing analysis: 2026-02-01*
