# Cove Code Quality Review — 2026-06-05 (Lens)

**Scope**: All 4 packages (server, plugin, client, shared) — ~4619 LOC reviewed file-by-file.
**Verdict**: Generally clean and well-organized. Repos and routes follow predictable patterns; shared types are minimal and Discord-accurate. The two real pain points are **the plugin's `channel.ts`** (one monster function doing dispatch orchestration) and **the server's `schema.ts`** (migrations + DDL + seeding fused together). A handful of smaller duplications are worth tidying.

---

## What's GOOD (don't touch)

- **`shared/`** (161 LOC): Tight, focused, well-commented. `snowflake.ts` is exemplary — clock-backward handling, increment wrap, all documented.
- **Repos** (`server/src/repos/*`): Each is a single class, ~50–115 LOC, one row-mapper, no clever abstractions. `ChannelsRepo`, `UsersRepo`, `ReadStatesRepo` are textbook.
- **`tool-progress.ts`**: Long-ish (222 LOC) but it's mostly a thin adapter over `plugin-sdk/channel-streaming`. Mechanical and readable.
- **Client stores** (zustand): Each store is ≤100 LOC, one slice of state. Good separation.
- **`gateway-client.ts`**: Clean WS client with heartbeating, exponential backoff, no surprises.
- **`validation.ts`**: 35 LOC, does exactly what it says. No DSL, no schema lib pulled in — appropriate for this size.

---

## Proposals

### 1. Decompose `plugin/src/channel.ts` — extract the dispatch pipeline
**What**: Break the 538-line file (one ~250-line `messageCreate` handler nested 5+ deep) into 3–4 focused modules.
**Where**: `plugin/src/channel.ts`
**Why**:
- The `messageCreate` handler does six concerns inline: per-channel abort tracking, typing keepalive, draft lifecycle, tool-progress tracking, runtime patching, and dispatch with timeout/abort racing. Reading it requires holding all six in mind at once.
- The `patchedRuntime` object literal nests `channel.reply.dispatchReplyWithBufferedBlockDispatcher → originalDispatcher → dispatcherOptions.deliver → replyOptions.{onPartialReply,onToolStart,…}` — every reviewer has to mentally track which `isCurrent()` belongs to which closure scope.
- Twelve nearly-identical `if (!isCurrent()) return; toolProgress.onX(payload)` callbacks could be table-driven or generated from a list.
- The `editQueue` Promise-chain + `sendOrEdit` lambda inside `messageCreate` is a piece of mutable state that should live in its own helper.

**How**: Suggested split:
- `plugin/src/dispatch-controller.ts` — owns `pendingDispatches` Map, registration/abort lifecycle (`registerDispatch`, `isCurrent`, `releaseDispatch`).
- `plugin/src/draft-streamer.ts` — `createDraftStreamer(restClient, channelId, log)` returning `{ update, seal, finalize, deleteDraft }` and owning the `editQueue`/`sendOrEdit`/draft-id state.
- `plugin/src/progress-wiring.ts` — `wireProgressCallbacks(toolProgress, draft, isCurrent)` that returns a single `replyOptions` fragment, collapsing the 12 callback handlers into one mapping.
- `channel.ts` (the plugin definition) shrinks to ~150 LOC: meta/capabilities/config + a `messageCreate` that orchestrates the three helpers above.

**Risk**: Refactor touches the hot path. Mitigation: existing tests should pass unchanged; add a focused test for `createDraftStreamer` and `createDispatchController` (both pure logic, easy to unit-test in isolation — currently they aren't reachable).
**Effort**: M (1 day)
**Priority**: P1

---

### 2. Split `server/src/db/schema.ts` — migrations per file
**What**: Move each `migrateVNtoVN+1` into its own file under `server/src/db/migrations/` and keep `schema.ts` for the runner + fresh-DB DDL only.
**Where**: `server/src/db/schema.ts` (627 LOC)
**Why**:
- The file mixes five very different concerns: migration registry/runner, V0→V1 legacy rename/column-add, V1→V5 schema changes, fresh-DB `createAllTables`, `initDb` entrypoint, and two `seed*` functions.
- `migrateV2ToV3` alone is 120+ lines and contains six independent ID-mapping passes — it's hard to scan because there are no section boundaries the IDE can navigate.
- `migrateLegacyToV1` + `migrateRenameTable` + `migrateChannelsToDiscordSchema` are ~100 LOC of historical cruft (renames `scenes → channels`, handles `position_x` columns). They're probably already dead code for any DB created after V1 shipped — but they're indistinguishable from active migrations in this file.

**How**:
```
server/src/db/
  schema.ts          (~150 LOC: runner, createAllTables, initDb, seed*)
  migrations/
    index.ts         (registry: { 1: m1, 2: m2, ... })
    v1-legacy.ts     (migrateV0ToV1 + legacy helpers — guarded by "if no tables exist" check)
    v2-read-states.ts
    v3-snowflake.ts
    v4-fk-constraints.ts
    v5-last-message-id.ts
    util.ts          (tableExists, hasColumn, addColumnIfMissing, isSnowflake)
```
**Bonus**: also evaluate whether `migrateLegacyToV1` (and its helpers `migrateRenameTable`, `migrateChannelsToDiscordSchema`) can be retired entirely — if no production DB still has `scenes` tables or `position_x` columns, drop ~100 LOC. If it must stay, isolate it so future readers can skip it.

**Risk**: Migration ordering matters — keep the `LATEST_VERSION = N` + numeric registry contract intact. Run the full test suite against an in-memory DB to confirm no behavior change.
**Effort**: M
**Priority**: P1

---

### 3. Remove redundant per-route `requireAuth` middleware
**What**: Delete `const auth = requireAuth(repos.users)` and per-route `auth` middleware applications in routes files; rely on the global `/api/*` middleware in `app.ts`.
**Where**: `server/src/routes/channels.ts`, `server/src/routes/agents.ts`
**Why**: `app.ts` already applies `requireAuth` globally to `/api/*` (except `PUBLIC_PATHS`). Every route in `channelRoutes` and `agentRoutes` is mounted under `API_PREFIX = "/api/v10"`, so the global middleware already runs. The per-route `auth` mw is dead — `c.get("botUser")` is already populated when the handler runs. The duplication makes a reader think there's something special about routes that have explicit `auth`, when in fact every route has it.

**How**: Remove the `auth` middleware from route definitions; delete the `const auth = requireAuth(repos.users)` lines.
**Risk**: Very low — the middleware was idempotent. Add an integration test asserting an unauthenticated request to a previously-explicit-`auth` route still 401s.
**Effort**: S (15 min)
**Priority**: P1

---

### 4. Extract a `requireSelfOr@me` helper for user-scoped routes
**What**: Three handlers in `agents.ts` (`PATCH /users/:id`, `DELETE /users/:id`, `POST /users/:id/token`) duplicate the same ~6-line block: resolve `@me`, check `id !== actorId → 403`, check `exists → 404`.
**Where**: `server/src/routes/agents.ts`
**Why**: Same pattern, three times, drift-prone (one of the three forgets the `exists` check). Pulling it into a tiny helper saves ~15 LOC and makes intent obvious.
**How**:
```ts
function resolveSelf(c: Context<AppEnv>, repos: Repos): string | Response {
  const raw = c.req.param("id");
  const actorId = c.get("botUser").id;
  const id = raw === "@me" ? actorId : raw;
  if (id !== actorId) return c.json({ message: "Missing Permissions", code: 50013 }, 403);
  if (!repos.users.exists(id)) return c.json({ message: "Unknown User", code: 10013 }, 404);
  return id;
}
```
Each handler becomes:
```ts
const id = resolveSelf(c, repos); if (id instanceof Response) return id;
```
**Risk**: Low.
**Effort**: S
**Priority**: P2

---

### 5. Collapse `MessagesRepo.list` branches; sender naming
**What**: Two small simplifications in `server/src/repos/messages.ts`.
**Where**: `server/src/repos/messages.ts`
**Why**:
- **(a)** `list()` has four `if/else` branches each building a similar SQL via string concatenation with `MSG_SELECT`. Could be expressed as a small `{ where, order, params }` builder, or — simpler — just inline the four branches but extract one helper `runMessageQuery(sql, ...params)` so the SELECT prefix isn't repeated 5 times.
- **(b)** The `Message` row resolution does `row.sender_username ?? row.sender_name ?? row.sender` — three fallbacks. `sender_name` is a denormalized column written at insert time (line 80) but never updated when the user renames. This is either a bug-in-waiting or `sender_name` is vestigial. Pick one: either (i) drop `sender_name` from the column list and rely on the JOIN, or (ii) document why the historical snapshot matters. As-is it's the worst of both: stored, sometimes stale, sometimes ignored.

**How**:
```ts
const runQuery = (sql: string, ...params: unknown[]) =>
  this.db.prepare(`${MSG_SELECT} ${sql}`).all(...params) as MessageRow[];
```
For (b): decide intent and prune one of the two name sources.
**Risk**: (a) zero. (b) needs a moment of thought — if `sender_name` exists to preserve the username at send-time (Discord-style "this message was sent when X was named Foo"), document that and remove the LEFT JOIN fallback. Otherwise drop the column in a V6 migration.
**Effort**: S
**Priority**: P2

---

### 6. Replace magic gateway opcodes in client `useWebSocketStore`
**What**: Client uses literal `op === 10`, `op: 2`, `op: 1`, `op === 11` instead of `GatewayOpcode` enum from `@cove/shared`.
**Where**: `client/src/stores/useWebSocketStore.ts:42–55`
**Why**: The server side imports and uses `GatewayOpcode.HELLO/IDENTIFY/HEARTBEAT/HEARTBEAT_ACK`. The shared enum exists for exactly this reason. Magic numbers in the client mean the next opcode change requires hunting greps.
**How**: `import { GatewayOpcode } from "@cove/shared"` and replace the four literals. Also: the `gatewayEvents` Set hoisted to the bottom of the file is fine but reads as misplaced — move it above the store definition.
**Risk**: Zero.
**Effort**: S (5 min)
**Priority**: P2

---

### 7. Remove `as any` casts in `channel.ts` plugin definition
**What**: Several `as any` casts in `plugin/src/channel.ts` — `id: "cove" as any`, `meta.id: "cove" as any`, `peer: { kind: "group" as any }`, `runtime: patchedRuntime as any`.
**Where**: `plugin/src/channel.ts`
**Why**: Each `as any` is a silent contract with the SDK that the type checker can no longer enforce. If `ChannelPlugin<CoveAccount>` typing genuinely doesn't allow `"cove"` as a channel id, that's either an SDK bug (fix it upstream) or the plugin is using the wrong generic. The `patchedRuntime as any` is the most worrying — it bypasses all typing on the runtime contract, so any SDK signature change here will fail silently at runtime.
**How**:
- Investigate `ChannelPlugin` type — likely `id` is a union and needs `as ChannelId` or a `declare module` augmentation.
- For `patchedRuntime`: declare an explicit `PatchedChannelRuntime` type matching what `dispatchInboundDirectDmWithRuntime` accepts and assign without cast.
- `peer.kind: "group"` — narrow to the SDK's actual peer-kind union.
**Risk**: Low; gains type-safety for the most fragile file in the codebase.
**Effort**: S–M (depends on SDK type cooperation)
**Priority**: P2

---

## Not Proposed (considered, rejected)

- **"Split routes/messages.ts (229 LOC)"** — it's 8 sibling handlers, each ~20 LOC, with no shared mutable state. Splitting would be ceremony. Keep.
- **"Use a query builder / Zod"** — package is small enough that `validation.ts` (35 LOC) plus hand-written SQL is *less* complexity than adding deps. Don't.
- **"Repo interface abstraction"** — `Repos` interface in `repos/index.ts` is already minimal and right-sized. Adding `IRepository<T>` would be over-engineering.
- **"Extract gateway dispatcher event broadcasting helpers"** — `dispatcher.ts` has many small methods, but each is ~5 lines and reads top-to-bottom. The clarity is worth the line count.

---

## Summary
| # | Title | Priority | Effort |
|---|---|---|---|
| 1 | Decompose `channel.ts` dispatch pipeline | P1 | M |
| 2 | Split `schema.ts` migrations per file | P1 | M |
| 3 | Remove redundant per-route `requireAuth` | P1 | S |
| 4 | Extract `resolveSelf` helper | P2 | S |
| 5 | Simplify `MessagesRepo.list` + clarify `sender_name` | P2 | S |
| 6 | Use `GatewayOpcode` enum in client WS store | P2 | S |
| 7 | Remove `as any` in plugin `channel.ts` | P2 | S–M |

Two P1s are the real signal — `channel.ts` and `schema.ts` are the only files that exceed what a single screen of reading comfortably holds. Everything else is good housekeeping.
