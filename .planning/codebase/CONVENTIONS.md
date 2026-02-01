# Coding Conventions

**Analysis Date:** 2026-02-01

## Naming Patterns

**Files:**
- React components: PascalCase (e.g., `AgentChatCard.tsx`, `AgentSendButton.tsx`)
- Utilities/hooks: camelCase (e.g., `useCodeTheme.ts`, `useFileChangeListener.ts`, `formatTimeAgo.ts`)
- Stores: kebab-case (e.g., `sub-chat-store.ts`, `changes-store.ts`)
- Routers: kebab-case with router suffix (e.g., `claude-code.ts`, `worktree-config.ts`)
- Test files: Not found (no test suite configured)

**Functions:**
- camelCase for all function names
- Async functions prefixed with clear intent: `fetch*`, `get*`, `create*`, `update*`, `delete*`
- Example: `getGitRemoteInfo()`, `createAppRouter()`, `getFileContents()`

**Variables:**
- camelCase for regular variables
- Prefix atom variables with `Atom` suffix (e.g., `selectedAgentChatIdAtom`, `agentsUnseenChangesAtom`)
- Constants in CONSTANT_CASE (implied but not heavily used)
- Boolean variables prefixed with `is`, `has`, `show`, `should` (e.g., `isLoading`, `hasUnseenChanges`, `showVoiceInput`, `shouldShowGlow`)

**Types:**
- PascalCase for interfaces and types
- Example: `AgentChatCardProps`, `AgentSubChatStore`, `SubChatMeta`
- Suffix with `Props` for React component prop interfaces
- No `I` prefix convention

## Code Style

**Formatting:**
- No explicit linter or formatter configured
- TypeScript strict mode enabled: `"strict": true`
- File extensions: `.ts` for utilities/config, `.tsx` for React components
- Indentation: 2 spaces (inferred from codebase)

**Linting:**
- No eslint or prettier configuration found
- Code is manually formatted and reviewed

**Type Safety:**
- TypeScript strict mode enforced
- All dependencies declare types via `@types/*` packages
- Zod schemas used for runtime validation in tRPC routers

## Import Organization

**Order:**
1. External packages: React, utilities, UI libraries
2. Internal modules: relative imports from shared logic
3. Type imports when needed (e.g., `import type { ... }`)

**Path Aliases:**
- `@/*` → `src/renderer/*` (renderer paths only)
- No aliases for main process

**Example from `agent-chat-card.tsx`:**
```typescript
import { useState, useCallback } from "react"
import { cn } from "../../../lib/utils"
import { GitHubLogo, IconSpinner, PlanIcon, AgentIcon } from "../../../components/ui/canvas-icons"
import { useAtomValue } from "jotai"
import { agentsUnseenChangesAtom, lastChatModesAtom } from "../atoms"
```

## Error Handling

**Patterns:**
- Throw descriptive Error objects with user-friendly messages
- Never throw raw strings
- Include context in error messages (file paths, operation names)
- Example: `throw new Error("Invalid GitHub URL or repo format")`
- Custom error types used for specific domains: `PathValidationError` in `src/main/lib/git/security/path-validation.ts`

**Error Messages:**
- Descriptive, user-readable text
- Include what failed and why
- Example: `throw new Error("Branch name too long (max 250 characters)")`

**No try-catch found in main git operations** - errors propagate and are handled at router/caller level

## Logging

**Framework:** Console (no third-party logging library)

**Patterns:**
- Prefix logs with context in brackets: `[GitWatcher]`, `[getFileContents]`, `[worktree-setup]`
- Use appropriate log level:
  - `console.log()` - Info/debug messages with context
  - `console.error()` - Error conditions
  - `console.warn()` - Warnings (non-fatal issues)
- Examples from codebase:
  ```typescript
  console.log("[getStatus] Cache hit for:", input.worktreePath)
  console.error("[GitHub] Repo schema validation failed:", result.error)
  console.warn(`[worktree] Setup completed with errors: ${setupResult.errors.join(", ")}`)
  ```

**No custom error tracking** - Sentry and PostHog disabled in open source builds unless env vars set

## Comments

**When to Comment:**
- Complex git operations (worktree isolation, stash handling)
- Non-obvious algorithm logic
- Integration points with external systems (GitHub API, OAuth)
- Migration logic (historical data transformation)

**JSDoc/TSDoc:**
- Used extensively in main process logic
- Parameters documented with `@param`
- Return values documented with `@returns`
- Example from `git-factory.ts`:
  ```typescript
  /**
   * @param worktreePath - Path to the git worktree/repository
   * @param timeout - Timeout in milliseconds (defaults to LOCAL timeout)
   * @returns Configured SimpleGit instance
   */
  ```

- React components use inline comment block for props documentation
- Example from `agent-send-button.tsx`:
  ```typescript
  interface AgentSendButtonProps {
    /** Whether the system is currently streaming */
    isStreaming?: boolean
    /** Whether the system is currently submitting/generating */
    isSubmitting?: boolean
  }
  ```

## Function Design

**Size:**
- Generally compact (50-200 lines) with some exceptions for complex workflows
- Long functions: git operations (`chats.ts` router ~1700 lines, but segmented by logical domains)
- Most utilities: 10-40 lines

**Parameters:**
- Use objects for multiple parameters (destructuring pattern)
- Example from `projects.ts`:
  ```typescript
  const result = await dialog.showOpenDialog(window, {
    properties: ["openDirectory", "createDirectory"],
    title: "Select Project Folder",
    buttonLabel: "Open Project",
  })
  ```

**Return Values:**
- Explicit return types in interfaces and tRPC procedures
- Nullable returns: `string | null` pattern used
- Array operations return arrays, not undefined

**Async Functions:**
- Consistently used for I/O operations
- No callback hell - uses await/async pattern throughout

## Module Design

**Exports:**
- Named exports: `export function`, `export const`, `export interface`
- No default exports used

**Barrel Files:**
- Not used extensively
- Each module imports directly from specific files

**Zustand Store Pattern:**
- Used for complex state (sub-chats, changes)
- Stores use `create()` factory
- Methods include state access via `get()` and setter via `set()`
- localStorage integration for persistence
- Example from `sub-chat-store.ts`:
  ```typescript
  export const useAgentSubChatStore = create<AgentSubChatStore>((set, get) => ({
    chatId: null,
    activeSubChatId: null,
    // ... state
    setChatId: (chatId) => {
      if (!chatId) {
        set({ chatId: null, activeSubChatId: null, ... })
        return
      }
      // ... logic
      set({ chatId, openSubChatIds, activeSubChatId, pinnedSubChatIds })
    },
  }))
  ```

**Jotai Atom Pattern:**
- Used for window-scoped UI state
- Atoms imported and used with `useAtomValue()`, `useSetAtom()`, `useAtom()`
- Location: `src/renderer/features/agents/atoms`, `src/renderer/lib/atoms`

**React Hooks Pattern:**
- Custom hooks prefixed with `use`
- Leverage Jotai atoms and React hooks
- Example: `useCodeTheme()` reads atoms and `useTheme()` from next-themes

**tRPC Router Pattern:**
- All routers in `src/main/lib/trpc/routers/`
- Procedures use: `publicProcedure.query()`, `publicProcedure.mutation()`
- Input validation via Zod schemas
- Context via tRPC context (includes `getWindow()`)
- Example from `projects.ts`:
  ```typescript
  export const projectsRouter = router({
    list: publicProcedure.query(() => { ... }),
    get: publicProcedure
      .input(z.object({ id: z.string() }))
      .query(({ input }) => { ... }),
    create: publicProcedure
      .input(z.object({ path: z.string(), name: z.string().optional() }))
      .mutation(async ({ input }) => { ... }),
  })
  ```

---

*Convention analysis: 2026-02-01*
