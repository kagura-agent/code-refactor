# Cove Code Quality Analysis

## Proposals

### 1. Refactor `plugin/src/channel.ts` (The Monolithic Dispatcher)
**What**: Extract `patchedRuntime` and dispatch logic from `messageCreate`
**Where**: `plugin/src/channel.ts`
**Why**: The `messageCreate` handler spans over 300 lines with 5+ levels of deep nesting. Inline definition of the `patchedRuntime` and overriding numerous callbacks inline creates high cognitive load and makes the file extremely hard to read, test, or extend.
**How**: Move the runtime patching, draft lifecycle management, and block dispatch overriding into separate helper functions (e.g., in a new file `plugin/src/cove-dispatch-runtime.ts`). 
**Risk**: Moderate. Care must be taken to preserve the `isCurrent` closure logic and `AbortController` signaling to avoid race conditions during streaming.
**Effort**: M
**Priority**: P1 (should fix)

### 2. De-clutter `server/src/db/schema.ts`
**What**: Move historical migrations out of the main schema file.
**Where**: `server/src/db/schema.ts`
**Why**: The file is 627 lines long, mostly filled with complex historical migration logic (like the V2->V3 UUID to Snowflake migration, which alone spans hundreds of lines). This obscures the actual current schema structure and makes the file unwieldy.
**How**: Create a `server/src/db/migrations/` directory. Keep the migration runner and schema definition in `schema.ts`, but extract `migrateV0ToV1`, `migrateV2ToV3`, etc. into their own imported files.
**Risk**: Very low, pure code organization.
**Effort**: S
**Priority**: P2 (nice to have)

### 3. DRY Route Handlers via Middleware
**What**: Extract repetitive channel/guild membership checks into a Hono middleware.
**Where**: `server/src/routes/messages.ts`, `server/src/routes/channels.ts`
**Why**: Almost every message route repeats the exact same 5 lines of boilerplate to verify the user has access to the channel via `requireGuildMember`. This violates DRY and clutters the core route logic.
**How**: Create a Hono middleware (e.g., `requireChannelAccess`) that performs this check and injects the resolved channel into the Hono context (`c.set('channel', ch)`). 
**Risk**: Low. Requires correctly typing the Hono environment variables.
**Effort**: S
**Priority**: P1 (should fix)
