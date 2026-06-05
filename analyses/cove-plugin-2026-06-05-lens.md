# Cove Plugin Layer — Refactor Analysis (Lens)

**Date**: 2026-06-05
**Scope**: `~/repos/forks/cove/packages/plugin/src/` (7 source files, 3 test files, 1739 LOC)
**Focus areas**: Discord-protocol fidelity, Gateway event coverage, reconnect robustness, single-guild assumption, dispatch pipeline correctness.

---

## Executive Summary

The plugin is a thin, working bridge but is **protocol-incomplete** and **stateless across reconnects**. It implements the bare minimum (READY + MESSAGE_CREATE), so any server-side improvement the refactor adds (MESSAGE_UPDATE, CHANNEL_*, GUILD_CREATE, TYPING_START, MESSAGE_DELETE_BULK, PRESENCE_UPDATE) is silently dropped. Reconnect re-IDENTIFYs from scratch — no `s` (seq) tracking, no `session_id` capture, no `RESUME` (opcode 6 is even *defined* in shared types but never sent or received). On any flap, every message in the gap is lost forever.

The dispatch pipeline in `channel.ts` is the strongest part of the file (per-channel `AbortController`, sequential `editQueue`, finalizable draft lifecycle, isCurrent guards) but it leaks subtle assumptions: single account, single guild, status never reset on close, `cleanupAndSend` skips `isCurrent()` re-check, `loadDirectDm` dynamic-imported per message.

REST client has **zero error policy** — no retries, no 429 handling, no fetch timeout, no idempotent-DELETE handling, no AbortSignal threading.

Below: 14 proposals, sorted by priority.

---

## P0 — Must-fix before refactor lands

### Gateway: track sequence + session_id, support RESUME
**What**: Implement Discord RESUME flow on the plugin side (op 6, plus session_id + last seq).
**Where**: `gateway-client.ts`, `types.ts`, `channel.ts` (no changes).
**Why**: Today every reconnect = new IDENTIFY = lost messages between disconnect and HELLO. The plugin already aborts pending dispatches on reconnect (good) but has no way to *replay* what it missed. Servers issue #224/#229 are matched by exact symmetry: the plugin must send `{op:6, d:{token, session_id, seq}}` after HELLO when it has prior state, and only fall back to IDENTIFY if it gets INVALID_SESSION (op 9) or has no session.

  Concrete gaps:
  - `GatewayOpcode` enum in `@cove/shared/types.ts` lacks `RECONNECT=7`, `INVALID_SESSION=9`. Plugin's `default: break` silently drops them.
  - `handleDispatch` stores `data.user` from READY but throws away `data.session_id`.
  - `handlePayload` never reads `payload.s`. A `lastSeq` field is missing.
  - Heartbeat sends `d: null`. Discord protocol requires `d: lastSeq` so the server can detect missed dispatches.
  - `reconnect` event is emitted on the *second* READY (`hasConnectedOnce`) — it should be emitted only when the server forces re-init (i.e. after INVALID_SESSION + fresh IDENTIFY). A successful RESUME should emit `resumed`, not `reconnect`, so the channel doesn't abort dispatches that survived the gap.

**How**:
1. Extend `GatewayOpcode` enum (RECONNECT, INVALID_SESSION, REQUEST_TYPING reserved — see #214 below).
2. Track `lastSeq: number | null` and `sessionId: string | null` on `CoveGatewayClient`. Update `lastSeq = payload.s` on every DISPATCH. Capture `sessionId` from READY.
3. On HELLO: if `sessionId && lastSeq` → send RESUME; else → IDENTIFY.
4. Handle INVALID_SESSION by dropping session/seq and re-IDENTIFY (with the documented small jittered delay).
5. Handle RECONNECT (op 7) by closing and reconnecting (preserving session/seq for RESUME).
6. Heartbeat payload `d` becomes `this.lastSeq`.
7. Add `resumed` event distinct from `reconnect`. In `channel.ts`, only abort pending dispatches on hard reconnect.
**Risk**: Behavioral change for existing users. If server doesn't yet implement RESUME, plugin must gracefully degrade (server replies INVALID_SESSION → fresh IDENTIFY).
**Test Impact**: New tests required (no current gateway-client tests exist). Existing tests unaffected.
**Effort**: M
**Priority**: P0 (mirrors server #224, #229; together they unlock zero-loss reconnect)

---

### Gateway: dispatch all server events, not just READY/MESSAGE_CREATE
**What**: Add a public event surface for every event the server can emit, even if the plugin no-ops most of them today.
**Where**: `gateway-client.ts`, `types.ts`.
**Why**: Server `dispatcher.ts` already emits MESSAGE_UPDATE, MESSAGE_DELETE, MESSAGE_DELETE_BULK, CHANNEL_CREATE/UPDATE/DELETE, GUILD_CREATE, TYPING_START, PRESENCE_UPDATE. The plugin's `default: break` makes them invisible. Two consequences:
  - Future plugin features (cache invalidation, edit/delete propagation, presence-aware dispatch) require touching `gateway-client.ts` again.
  - Bugs are silent — there's no "unknown event" log, so a typo in `t` (server side) is undetectable.
**How**:
- Generic `dispatch` event with `{t, d, s}` for forward-compat, **and** typed events per known kind.
- Log unknown `t` at debug level so we can see what's being ignored.
- Move event-name strings to a `GatewayEvent` enum in `@cove/shared`.
**Risk**: Low — adds surface, doesn't change behavior.
**Test Impact**: None broken; new tests should assert known events fire and unknown events log.
**Effort**: S
**Priority**: P0 (foundation for everything else)

---

### REST: retry, 429 awareness, fetch timeout
**What**: Wrap `fetch` with retry/backoff, Retry-After honoring, AbortSignal, and timeout.
**Where**: `rest-client.ts` (full rewrite of `request`).
**Why**: Today a single 429 from the server (or any 5xx) tears down an entire dispatch. The plugin sends `editMessage` per stream tick (throttled 250ms but unbounded over a long reply) — easy to trip rate limits. Also `fetch` has no timeout: if the server hangs, the streaming draft pipeline hangs forever (the per-channel 120s dispatch timeout *eventually* fires but only if upstream is racing it).

  Concrete gaps:
  - No `AbortSignal` plumbed into fetch; the abort controller in `channel.ts` never reaches the wire.
  - 429 path does `throw new Error(...)`; should sleep `Retry-After` then retry.
  - 5xx is non-retryable today (single `throw`).
  - DELETE 404 is treated as failure; should be idempotent success.
  - `sendTyping` silently swallows all errors (`catch(() => {})`) — even a 401 disappears.

**How**:
- `request<T>(method, path, body, {signal, idempotent}): Promise<T>` with: 1× retry on network error, retry on 429/502/503/504 with exponential backoff capped by `Retry-After` header.
- Pass `ctx.abortSignal`-derived signal from `channel.ts` so destroy() actually cancels in-flight HTTP.
- Add `requestTimeoutMs` (default 15s) via `AbortSignal.timeout`.
- DELETE: treat 404 as success.
- `sendTyping`: at least log warn at first failure, drop subsequent for the channel within 5s window.
**Risk**: Retries can amplify load if server is genuinely down — cap retry count and total time.
**Test Impact**: New tests for `rest-client.ts` (no current tests). No existing tests touch REST directly.
**Effort**: M
**Priority**: P0

---

### Multi-guild + multi-account: stop hardcoding "cove"
**What**: Remove single-guild assumption; index dispatch state by `(accountId, guildId, channelId)`.
**Where**: `channel.ts` (`pendingDispatches`, `getRestClient`, `resolveAccount`), `rest-client.ts` (`getChannels` default), `types.ts` (`CoveAccount`).
**Why**: Plugin assumes one Cove "guild" called `"cove"` and one OpenClaw account called `"default"`. Several places nail this in:
  - `resolveAccount` reads `cfg.channels?.["cove"]` (channel id literal). `accountId` parameter is accepted but ignored.
  - `listAccountIds: () => ["default"]` — only one account.
  - `CoveAccount.guildId` defaults to `"cove"` and is never used to scope channels.
  - `CoveRestClient.getChannels(guildId = "cove")` — default arg.
  - `pendingDispatches` keyed by `channelId` alone — two accounts joining the same channel id would collide. Also no `guild_id` scoping when Cove eventually multi-tenants.
  - `restClients` map cached by `baseUrl+token` is process-global; never cleared on plugin shutdown — leaks across hot-reload.
**How**:
- Drop the `"cove"` literal from `getChannels`; require explicit `guildId`.
- `resolveAccount` should take `accountId` seriously (read `cfg.channels.cove.accounts[accountId]` or fall back to the legacy single-account shape with deprecation warning).
- Discover guilds from READY (server should include `guilds: Guild[]` in the READY payload — symmetry with #228 GuildStore on the server).
- `pendingDispatches: Map<string, AbortController>` → key = `${guildId}:${channelId}` (or `${accountId}:${guildId}:${channelId}` if multiple accounts share a runtime).
- Clear `restClients` cache on `ctx.abortSignal` abort.
**Risk**: Breaking change for existing config users — keep backward-compat path (resolve old `channels.cove.token` shape into the first account).
**Test Impact**: `dispatch-resilience.test.ts` reconnect-cancels test uses `"channel-1"`/`"channel-2"` keys; format change requires updating helpers.
**Effort**: M
**Priority**: P0 (mirrors server #228; without it, plugin can't follow server when it grows past one guild)

---

## P1 — Important, do soon

### Gateway: opcode-4 collision avoidance (#214 mirror)
**What**: Reserve a non-Discord opcode for the plugin→server "I am typing" path *or* delete the Gateway-side typing entirely; never repurpose Discord op 4 (VOICE_STATE_UPDATE).
**Where**: `gateway-client.ts`, `@cove/shared` enum.
**Why**: Plugin currently uses REST `POST /channels/:id/typing` (good — no collision today). But there's no documented rule preventing a future contributor from adding a Gateway `op:4` shortcut and tripping #214. Lock it in:
  - Add `RESERVED_VOICE_STATE_UPDATE = 4` to the enum with a comment "DO NOT USE — Discord reserved".
  - If Cove ever wants gateway-side typing, allocate a high opcode (e.g. 100+) outside Discord's range and prefix events accordingly.
**Risk**: Trivially low.
**Test Impact**: None.
**Effort**: S
**Priority**: P1

---

### Reconnect: state re-sync, status updates, deferred work
**What**: On reconnect, refresh channel/guild caches and update plugin status (`connected:false` on close, `true` on ready).
**Where**: `channel.ts` (gateway event handlers).
**Why**:
  - On `close`, plugin logs but never calls `ctx.setStatus({connected:false})`. The OpenClaw UI thinks Cove is up while it's flapping.
  - After RESUME (when implemented), no-op is correct. After hard reconnect (new session), plugin should re-fetch channel list (in case a CHANNEL_CREATE happened during the gap) and clear any cached message ids the dispatcher may hold.
  - Right now, the *only* reconnect side-effect is "abort pending dispatches" — which is half a recovery, not a full one.
**How**:
- `gatewayClient.on("close")` → `ctx.setStatus({connected:false, running:true, ...})`.
- `gatewayClient.on("ready")` after first → call a `resyncState()` helper that re-`getChannels` and clears any per-channel state that's safe to discard.
- Distinguish `resumed` (no-op) vs `reconnect` (full resync).
**Risk**: Resync can be expensive if guild has many channels — gate behind a feature flag or do incremental.
**Test Impact**: None broken; add test for status transitions.
**Effort**: S
**Priority**: P1

---

### `cleanupAndSend` re-checks `isCurrent()`
**What**: Re-check `isCurrent()` after `await deleteMessage` and before `sendMessage` in `cleanupAndSend`.
**Where**: `channel.ts:84-101`.
**Why**: `cleanupAndSend` is reached on the slow-path final-edit failure. Between `await deleteMessage` and `await sendMessage`, a new message can arrive on the same channel and abort this dispatch. Result: a stale final reply is sent after abort. Every other side effect inside the dispatcher gates on `isCurrent()`; this helper is the one place that doesn't.
**How**: Pass `isCurrent: () => boolean` into `cleanupAndSend` and short-circuit between the two awaits.
**Risk**: None.
**Test Impact**: Add a unit test for the race; existing tests unaffected.
**Effort**: S
**Priority**: P1

---

### Hoist `loadDirectDm` import; remove dynamic import on hot path
**What**: Replace `const loadDirectDm = () => import("openclaw/plugin-sdk/channel-inbound")` + per-message await with a top-level static import.
**Where**: `channel.ts:14, 256`.
**Why**: Every inbound message pays one dynamic-import cycle. The comment on the line offers no rationale (it predates the rest of the file). Static import is faster, easier to type, and lets `dispatchInboundDirectDmWithRuntime` be tree-shaken/typed.
**Risk**: If there *was* a circular-dep reason, static import would fail at module-load. Try and see; revert with a comment if it does.
**Test Impact**: None expected.
**Effort**: S
**Priority**: P1

---

### IDENTIFY payload: include `intents`/`properties` placeholders
**What**: Send a Discord-shaped IDENTIFY (`{token, intents, properties}`) even if the server ignores most fields today.
**Where**: `gateway-client.ts:sendIdentify`.
**Why**: Discord-compat means the plugin should be a drop-in for a real Discord bot library if Cove ever pivots. Empty `intents:0` and `properties:{os,browser,device}` cost nothing now, save a refactor later.
**Risk**: Server must tolerate extra fields (almost certainly does — it pulls `token` only).
**Test Impact**: None.
**Effort**: S
**Priority**: P1

---

### Use `GET /api/v10/gateway` for WS URL discovery
**What**: Call `restClient.getGatewayUrl()` instead of building the URL by `replace(/^http/, "ws")`.
**Where**: `channel.ts:222`.
**Why**: The REST helper `getGatewayUrl` already exists but is unused. Discord's pattern is to discover the gateway URL via REST so the server can return shard info, session-start limits, regional URLs, etc. Today the plugin manually rewrites scheme + appends `/gateway`, which works only because the server happens to colocate REST and WS on the same host.
**Risk**: Adds one REST call before connect; if it fails, the plugin can't start. Cache the result and fallback to URL construction on error.
**Test Impact**: None broken; mock the endpoint in new tests.
**Effort**: S
**Priority**: P1

---

## P2 — Nice to have / hygiene

### Test coverage gaps
**What**: Add unit tests for `gateway-client.ts` (heartbeat ack zombie detection, reconnect backoff, RESUME flow once implemented) and `rest-client.ts` (429 retry, 404-on-DELETE idempotence, timeout).
**Where**: New test files.
**Why**: Three test files cover dispatch resilience, edit queue, and tool progress — but the network surface (gateway + REST) has zero direct tests. Both have known bugs above; tests will lock in fixes.
**Effort**: M
**Priority**: P2

---

### `outbound.sendText` unawareness of streaming drafts
**What**: When a tool calls `outbound.sendText` to the same channel that's mid-stream, the new message lands between draft edits and confuses the conversation.
**Where**: `channel.ts:200-209`.
**Why**: `sendText` goes straight to `restClient.sendMessage` without consulting `pendingDispatches` or `editQueue`. A plugin user calling outbound while a dispatch is streaming will produce out-of-order chat.
**How**: Either queue outbound through `editQueue` for that channel, or document that outbound is for non-streaming channels only.
**Risk**: Behavior change; document in plugin README.
**Effort**: S
**Priority**: P2

---

### DM vs channel routing
**What**: `messageCreate` always treats peers as `kind: "group"`; Discord-shape DMs (no `guild_id`) are not distinguished.
**Where**: `channel.ts:308-314` (`peer: { kind: "group", id: channelId }`).
**Why**: `capabilities.chatTypes: ["direct", "channel"]` claims DM support but the dispatch hard-codes group. If the server adds Discord-style DM channels (channel.type=1, no guild_id), the plugin will route them as group and pick the wrong DM policy / session key.
**How**: Inspect `message.guild_id` (or `channel.type` via cache) to choose `kind: "direct" | "group"`. Plumb sender_id correctly for DMs.
**Risk**: Touches security policy resolution — test with `dmPolicy:"open"` and `"closed"`.
**Effort**: M
**Priority**: P2

---

### `restClients` cache lifecycle
**What**: Process-global `restClients` Map is never cleared. Plugin shutdown leaks the entries; on hot-reload the new module-instance starts empty but the old one lingers as long as the runtime keeps the reference.
**Where**: `channel.ts:67-77`.
**Why**: Minor leak; matters in dev hot-reload and multi-tenant gateways.
**How**: Tie the cache to `ctx` (one cache per plugin runtime), or clear on `ctx.abortSignal` abort.
**Effort**: S
**Priority**: P2

---

## Cross-cutting Notes

- **Symmetry with server**: The four P0 items map almost 1:1 with server issues #214, #224, #228, #229. Land them as paired PRs (server change first, plugin follows) so seq/RESUME/guild discovery have a counterparty.
- **Discord compat is the contract**: Every deviation costs you the ability to swap in a real Discord library for testing. Each `default: break`, each `replace(/^http/, "ws")`, each hardcoded `"cove"` literal is a small future bug.
- **Tests are an asset for the hot path, a liability for the protocol**: 611 lines testing dispatch internals; 0 lines testing the wire. Invert.
- **Strong points worth preserving**: `createAbortableDispatch`, `editQueue`, `createFinalizableDraftLifecycle` integration, `isCurrent()` reference-equality guards, heartbeat ACK zombie detection, exponential reconnect backoff. These are good and should not be touched in the refactor.
