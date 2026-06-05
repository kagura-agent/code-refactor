# Cove Code Quality Review — Arch

Repo: `kagura-agent/cove`
Scope: all non-test source in `packages/server`, `packages/client`, `packages/plugin`, `packages/shared`
Date: 2026-06-05
Focus: cleanliness, simplicity, maintainability, over-engineering — not feature gaps.

## Overall read

Cove is generally clean for its size. Most files are small, direct, and easy to scan. The package boundaries are sensible:

- `shared` is tiny and stable: types + snowflakes only.
- server repos are simple data mappers over SQLite, with straightforward routes.
- client stores are intentionally thin Zustand stores.
- plugin REST/gateway clients are modest and readable.

The biggest maintainability risks are concentrated in a few places rather than spread throughout the codebase:

1. `packages/plugin/src/channel.ts` is doing too many jobs at once and contains the only truly hard-to-follow nested control flow.
2. `packages/server/src/db/schema.ts` is long because it preserves migration history; this is not inherently bad, but mixed migration utilities + current schema + seed code make it harder to review safely.
3. server routes repeat auth/membership/error boilerplate enough that behavior can drift.
4. client UI components are mostly fine, but several combine view, inline styles, data fetching, and side effects in ways that are starting to make files bulky.

I would not do a broad architectural rewrite. The code mostly benefits from extraction of local helpers and consistency cleanup, not new frameworks or heavy abstractions.

---

### Split the Cove plugin channel pipeline

**What**: Break `channel.ts` into focused modules and flatten the inbound dispatch flow.

**Where**: `packages/plugin/src/channel.ts` primarily; likely new files such as:
- `packages/plugin/src/account.ts` for `resolveAccount`, REST client cache
- `packages/plugin/src/dispatch-control.ts` for timeout/abort helpers
- `packages/plugin/src/streaming-reply.ts` for draft lifecycle + `sendOrEdit` + final delivery
- `packages/plugin/src/runtime-patch.ts` for the patched OpenClaw runtime/reply dispatcher

**Why**: `channel.ts` is 538 lines and mixes account resolution, REST client caching, gateway lifecycle, pending dispatch tracking, OpenClaw runtime patching, streaming draft lifecycle, tool progress forwarding, timeout handling, and shutdown cleanup. The `messageCreate` handler nests async setup, draft state, queueing, patched runtime construction, dispatch execution, error mapping, and cleanup in one function. This is the only file where local reasoning is genuinely expensive: reading a final reply path requires holding `pendingDispatches`, `abortController`, `draftState`, `draftMessageId`, `lastSentText`, `editQueue`, `typingCallbacks`, `toolProgress`, `patchedRuntime`, and two nested try/catch/finally blocks in mind.

**How**:
- Keep `channel.ts` as the plugin composition/root: account setup, gateway event registration, and high-level lifecycle only.
- Extract pure-ish helpers with explicit input/output contracts:
  - `createDispatchControllerRegistry()` or small helpers for per-channel abort/supersede/current checks.
  - `createStreamingReplyAdapter({ restClient, channelId, log, isCurrent, toolProgress })` returning `replyOptions`, `dispatcherOptions`, `seal/cleanup` hooks.
  - `patchRuntimeForCove({ channelRuntime, targetAgent, typingCallbacks, deliver, replyOptions })`.
- Use guard clauses in `messageCreate` and move the dispatch body into `handleIncomingMessage(ctx, account, message, deps)` so the event listener is short.
- Avoid adding a class unless state really needs identity; closures are enough here.

**Risk**: Medium. This is delicate code around stale dispatches and streaming edits. A careless extraction could reintroduce race conditions (the file has comments showing those were already fixed). Refactor with characterization tests around abort supersede, timeout, final edit fallback, and stale dispatch no-op behavior before changing behavior.

**Effort**: L

**Priority**: P1

---

### Split database migrations from current schema and seed logic

**What**: Separate migration history, schema creation, and seed helpers so `schema.ts` stops being a catch-all database module.

**Where**: `packages/server/src/db/schema.ts` (627 lines).

**Why**: The length is mostly legitimate: migrations are historical and should stay explicit. The problem is not simply line count; it is that one file contains:
- migration registry and runner
- legacy migration helpers
- current table creation DDL
- database initialization and FK validation
- seed channel/user helpers

That makes safe review harder. Current schema changes are visually interleaved with legacy compatibility code, and unrelated edits touch the same large file. The file also uses local helpers (`hasColumn`, `safeQuery`, recreate-table snippets) inside specific migrations, which is good for locality, but the overall module has too many responsibilities.

**How**:
- Split into:
  - `db/migrate.ts`: `LATEST_VERSION`, `migrations`, `runMigrations`, migration functions.
  - `db/schema-ddl.ts`: `createAllTables`, `addColumnIfMissing` if shared.
  - `db/seed.ts`: `seedChannels`, `seedUsers`, seed constants.
  - `db/index.ts` or keep `schema.ts`: public `initDb()` that wires them together.
- Preserve migration function names and order exactly. Do not compress or “clean up” old migrations unless tests prove equivalence.
- Consider adding a short header in `migrate.ts`: old migrations are historical snapshots; new migrations append only.

**Risk**: Low to Medium. Mostly mechanical, but migration code is high consequence. Risk is import cycles or accidentally changing migration order/visibility. Run existing DB migration tests and at least one fresh DB initialization after the split.

**Effort**: M

**Priority**: P1

---

### Centralize server route guard and Discord error responses

**What**: Replace repeated guild/member/channel guard blocks and literal error JSON with small route helpers.

**Where**:
- `packages/server/src/routes/channels.ts`
- `packages/server/src/routes/messages.ts`
- `packages/server/src/routes/agents.ts`
- `packages/server/src/app.ts` presences endpoint
- possibly `packages/server/src/routes/helpers.ts`

**Why**: Routes are readable, but there is significant copy-paste:
- `Unknown Guild` / code `10004`
- `Unknown Channel` / code `10003`
- `Unknown Message` / code `10008`
- `Missing Permissions` / code `50013`
- “guild exists + actor is member” repeated in multiple routes
- “load channel + actor is member” repeated already partially handled by `requireGuildMember`

This is not over-engineering yet, but it is a drift risk. For example, changing how membership failures are represented would require many edits. The current `helpers.ts` is a good start but too narrow.

**How**:
- Add tiny response helpers, not a framework:
  - `discordError(c, kind)` or constants like `unknownGuild(c)`, `unknownChannel(c)`, `missingPermissions(c)`.
  - `requireGuildAccess(repos, guildId, userId): boolean`.
  - `requireChannelAccess(repos, channelId, userId): Channel | null` (current `requireGuildMember` with a clearer name).
- Keep validation local to routes; do not build a generic router abstraction.
- Use helpers only where they remove repeated 3-5 line blocks.

**Risk**: Low. Main risk is hiding route-specific behavior behind helpers that become too clever. Keep helpers return-value based and explicit.

**Effort**: S

**Priority**: P1

---

### Extract bulky client UI sections and reusable display helpers

**What**: Split the largest client components where one file combines multiple UI sections, inline styles, and side effects.

**Where**:
- `packages/client/src/components/SettingsPanel.tsx` (285 lines)
- `packages/client/src/App.tsx` (257 lines)
- `packages/client/src/components/MessageItem.tsx` (172 lines)
- optionally `packages/client/src/components/MessageList.tsx` (152 lines)

**Why**: The client is still understandable, but the largest files are drifting toward “everything in one component.” `SettingsPanel.tsx` contains nav data, theme swatch, three section components, modal shell, profile/sidebar UI, and all styles. `App.tsx` contains visual viewport behavior, AntD theme config, login/invite pages, auth bootstrap, channel bootstrap, websocket setup, mobile layout state, and main layout. `MessageItem.tsx` duplicates body rendering/style between grouped and ungrouped messages and has a tiny redundant wrapper (`avatarColor()` just calls `pickAvatarColor()`).

This is not a call for a design system. The current inline-style approach is simple and consistent. The issue is file-level scan cost and repeated local JSX/styling blocks.

**How**:
- For `SettingsPanel.tsx`: move `ThemeSwatch` and section components to either `SettingsSections.tsx` or small sibling files; keep the modal shell in `SettingsPanel.tsx`.
- For `App.tsx`: extract `LoginPage`, `InviteCodePage`, `useAuthBootstrap`, and `useCoveSessionBootstrap` so the default export reads like layout state + render.
- For `MessageItem.tsx`: extract `MessageBody` and `AvatarCircle`; remove `avatarColor()` wrapper; share body style once.
- Do not move every `styles` object globally; colocated styles are fine when the component is small.

**Risk**: Low. Mostly mechanical React extraction. Watch for hook rules if extracting bootstrap logic; hooks should remain hooks, not plain helper functions.

**Effort**: M

**Priority**: P2

---

### Replace `any` seams in the plugin with narrow local adapter types

**What**: Reduce untyped `any` usage around the OpenClaw plugin integration by defining local structural types for the subset Cove actually uses.

**Where**: Mostly `packages/plugin/src/channel.ts`; small amount in `packages/plugin/src/gateway-client.ts` typed emitter.

**Why**: `channel.ts` has many `any` casts and payloads (`cfg`, `ctx`, `channelRuntime`, route params, dispatcher params, payload callbacks, log errors). Some are unavoidable because the upstream plugin SDK may not export all exact types, but the current volume makes refactors unsafe: TypeScript cannot help if `reply.dispatchReplyWithBufferedBlockDispatcher`, `routing.resolveAgentRoute`, or event payload shapes change. This compounds the complexity of the long nested dispatch pipeline.

**How**:
- Define narrow interfaces for only the members used:
  - `CovePluginConfigSection`
  - `RuntimeWithChannelRuntime`
  - `ChannelRuntimeSubset`
  - `ReplyDispatcherParams`
  - `PartialReplyPayload`, `ToolStartPayload`, etc. if SDK types are unavailable.
- Replace catch variables typed as `any` with `unknown` + helper `errorMessage(err)`.
- Keep unavoidable boundary casts at the edge, e.g. one cast when reading `ctx.channelRuntime`, instead of throughout the dispatch body.
- In `gateway-client.ts`, the typed emitter trick is acceptable; no need to overwork it unless it blocks strictness.

**Risk**: Low to Medium. Types may expose real shape mismatches. Avoid spending days modeling the entire SDK; the goal is to type Cove’s used subset.

**Effort**: M

**Priority**: P2

---

### Remove accidental public/stateful seams in client stores

**What**: Hide module-level mutable implementation details and reset cached API state consistently on logout.

**Where**:
- `packages/client/src/lib/api.ts`
- `packages/client/src/stores/useUserStore.ts`
- `packages/client/src/stores/useTypingStore.ts`
- `packages/client/src/lib/gateway-subscriptions.ts`

**Why**: A few small globals are pragmatic, but some are leaking across boundaries:
- `api.ts` has `_guildId` cache and a `resetGuildId()` function, but `useUserStore.logout()` does not call it. That is a small stale-state trap after account switching.
- `typingTimeoutIds` is exported from the store solely so subscription teardown can clear timers. That exposes store internals and makes typing lifecycle split across two files.
- `gateway-subscriptions.ts` stores handlers as `any` and clears typing timers externally.

None of this is severe, but the pattern makes cleanup behavior easy to miss.

**How**:
- Make logout/session reset a single helper, e.g. `resetClientSessionState()` that clears token, cached guild ID, websocket, stores that need clearing, and typing timers.
- Move typing timeout management behind store actions such as `setTyping(channelId, user)` and `clearAllTyping()`; stop exporting `typingTimeoutIds`.
- Make `gateway-subscriptions` keep type-safe unsubscribe callbacks instead of `{ event, handler: any }`, or have `dispatcher.on()` return an unsubscribe function.

**Risk**: Low. Risk is mostly changing logout/reconnect behavior unintentionally. Verify login → logout → login as another user, and websocket teardown/setup.

**Effort**: S

**Priority**: P2

---

### Keep the simple parts simple

**What**: Avoid refactoring the already-clean small modules just because they are plain.

**Where**:
- `packages/shared/src/*`
- server repos (`packages/server/src/repos/*.ts`)
- small client stores and components (`StatusDot`, `TokenDisplay`, `BotCreateForm`, `BotManagement`, `UserBar`, etc.)
- `packages/plugin/src/rest-client.ts`
- `packages/plugin/src/gateway-client.ts` except minor typing if desired

**Why**: Several files are “boring” in the best way: direct data mapping, small wrappers, or simple UI. Adding generic repositories, schema builders, service layers, or a global component/style abstraction would make the code more abstract without improving clarity. The current repo classes are repetitive but transparent; the client stores are small enough that local duplication is cheaper than a meta-store layer.

**How**:
- Prefer local helpers over new architectural layers.
- Do not introduce an ORM or migration framework unless SQLite migration complexity grows substantially beyond the current file split.
- Do not centralize all inline styles into a theme system; only extract styles/components when a file is bulky or duplicated.
- Keep `shared` minimal.

**Risk**: Very low. This is a restraint recommendation.

**Effort**: S

**Priority**: P2

---

## Notes on dead code / commented-out code

I did not find obvious large dead modules or commented-out code blocks in the source. The notable markers are mostly legitimate future-work comments:

- `packages/server/src/routes/messages.ts`: permission TODOs for delete/bulk-delete.
- `packages/server/src/ws/dispatcher.ts`: DM broadcast TODO.
- `packages/server/src/repos/guilds.ts`: clearly marked temporary default-guild behavior.

These are feature/product debt rather than cleanliness problems. For this review, I would not remove them unless their corresponding issues are closed or obsolete.

## Quick file-level cleanliness summary

- Good: `shared` is concise and cohesive.
- Good: server repos are direct and readable; no unnecessary generic data layer.
- Good: server routes are explicit; duplication is the main problem, not hidden complexity.
- Good: plugin `rest-client.ts`, `gateway-client.ts`, and `tool-progress.ts` are reasonably focused.
- Needs attention: plugin `channel.ts` is the highest-complexity file by far.
- Needs attention: database `schema.ts` should be split by responsibility, but migration verbosity itself is acceptable.
- Acceptable but growing: client `App.tsx`, `SettingsPanel.tsx`, `MessageItem.tsx`, `MessageList.tsx`.
