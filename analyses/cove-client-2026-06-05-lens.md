# 🔬 Lens — Cove Client State Management Audit

**Scope**: `packages/client/src/stores/*` + `packages/client/src/lib/{api,gateway-dispatcher,gateway-subscriptions}.ts` + `types.ts`
**Comparison target**: Discord client cache model (`Guild → Channel → Message`, READY snapshot, incremental Gateway events, resume/replay on reconnect).
**Verdict**: The slice is small and clean (≈600 LoC for state/gateway/api), but it bakes in a *single-guild, single-session, no-replay* worldview. Functionality works today; multiple foreseeable changes (multi-guild, member list, reconnect after sleep, optimistic send) require structural change, not additive patches.

---

## Discord-model gap table (factual)

| Concern | Discord client cache | Cove client today | Gap |
|---|---|---|---|
| Guild cache | `GuildStore` keyed by guild_id; carries channels/members/roles | `getGuildId()` caches single ID in a module var (`api.ts:7`); no `useGuildStore` | Multi-guild blocker |
| Channel cache | Nested under guild | Flat `channels: Channel[]` (`useChannelStore.ts`) | No `guild_id` partitioning |
| Member cache | `GuildMemberStore` keyed by (guild_id,user_id); populated by `GUILD_MEMBERS_CHUNK` | `BotStore.fetchBots` filters members; only bots survive; humans discarded | No member store; presence keyed to unknown users |
| User cache | Global `UserStore` (id → User) | Only `useUserStore` for *self*; message authors live inline on each Message | Duplication, no central rename propagation |
| READY snapshot | Populates guilds/channels/members/read_state/presences/user | READY only fills `presences` + `read_state` (`gateway-subscriptions.ts:78-86`); channels/me fetched separately via REST | READY is half-used |
| Reconnect | RESUME with `session_id`+`seq`; server replays missed events; fallback re-IDENTIFY | Re-IDENTIFY only; no session_id, no seq tracking, no re-fetch of channels/messages on reconnect (`useWebSocketStore.ts:74-86`) | Reconnect blindness |
| Optimistic send | Local insert with `nonce`, reconciled by `MESSAGE_CREATE` echo | `sendMessage` REST → wait → store update via Gateway event | No optimistic UX; perceived latency |
| Cache invalidation on channel delete | Cascades messages/typing/read_state | `removeChannel` clears active id only; `messages[channelId]` / `readStates[channelId]` / `typingUsers[channelId]` leak | Memory + stale state |
| Heartbeat ACK | Tracks last ACK; zombie detection forces reconnect | Op 11 silently ignored (`useWebSocketStore.ts:64`); no zombie detection | Half-open sockets can wedge |
| Event coverage | GUILD_*, MEMBER_*, USER_UPDATE, MESSAGE_REACTION_*, CHANNEL_PINS_UPDATE | Only the 10 listed in `gatewayEvents` set | Future events silently dropped (no warning) |

---

## Proposals

### 1. Introduce `useGuildStore` and key channels/members/messages by guild_id
**What**: Replace flat `channels: Channel[]` with `Map<guildId, Channel[]>` (or normalized `channelsById` + `channelIdsByGuild`); add a `useGuildStore` carrying `{guilds, activeGuildId}`; remove `_guildId` module cache from `api.ts`.
**Where**: `stores/useChannelStore.ts`, new `stores/useGuildStore.ts`, `lib/api.ts`, `lib/gateway-subscriptions.ts` (CHANNEL_* needs `guild_id`), every component that calls `useChannelStore`.
**Why**: Today a second guild would silently overwrite the cached `_guildId`; `channels` list would mix guilds; `activeChannelId` has no guild context. This is the single biggest structural debt. Even *before* multi-guild ships, the shape makes `GUILD_CREATE`/`GUILD_DELETE` impossible to wire without touching every store.
**How**:
1. Add `guild_id` to `Channel` consumers (already on the wire — verify in `@cove/shared`).
2. `useGuildStore` holds `guilds: Record<string, Guild>`, `activeGuildId`. Hydrated from `GET /users/@me/guilds` once at READY (or from READY payload if server adds it).
3. `useChannelStore` keyed by guild: `channels: Record<guildId, Channel[]>`; selectors `getChannelsForGuild(guildId)`.
4. Components read `useGuildStore(s=>s.activeGuildId)` and pass through.
**Risk**: Touches most UI components; selector churn could cause re-render storms if not memoized.
**Test Impact**: Every store test + gateway-subscriptions test needs guild_id fixtures. New: guild store tests, multi-guild dispatch test.
**Effort**: L
**Priority**: P1 (P0 if multi-guild is on the roadmap this quarter)

---

### 2. Use READY as the canonical snapshot — populate channels, self, guilds, members in one shot
**What**: Move channel-list, self-user, and member-list initialization from scattered REST calls into the `READY` handler, driven by READY payload.
**Where**: `lib/gateway-subscriptions.ts` (READY handler), `lib/gateway-dispatcher.ts` (extend `GatewayEventMap.READY`), `stores/useChannelStore.ts`, `stores/useUserStore.ts`, new `useMemberStore`.
**Why**: Today the client does READY → REST `/users/@me/guilds` → REST `/guilds/:id/channels` → REST `/guilds/:id/members` (lazy, for bots). That's 3 round-trips after the WS is already open and the server already knows what to send. Discord ships everything in READY (and `READY_SUPPLEMENTAL`) precisely because the cache is unusable until populated. Cove will keep growing READY-time dependencies (member list, presence, read_state, pinned messages) — centralize now.
**How**: Server emits `{user, guilds:[{id,name,channels:[...]}], read_state, presences, members?}`. Client READY handler hydrates all stores atomically; REST fallbacks remain for refresh.
**Risk**: Couples client/server payload shape. Requires server change (out of scope here but worth noting in proposal).
**Test Impact**: gateway-subscriptions READY test grows; remove channel-fetch effect from App component tests.
**Effort**: M (client side) + server work
**Priority**: P1

---

### 3. Wire reconnect state recovery (re-fetch on resume, or implement RESUME/seq tracking)
**What**: On WebSocket reconnect after a gap, either (a) emit `RESUME` with `session_id`+`seq` and have server replay, or (b) cheap path: re-trigger READY-equivalent snapshot fetch (channels, messages for active channel, presence, read_state).
**Where**: `stores/useWebSocketStore.ts` (track `sessionId`, `lastSeq`; send op 6 RESUME), `gateway-subscriptions.ts` (READY vs RESUMED branching), server gateway (out of scope but listed).
**Why**: Right now if the laptop sleeps for 5 minutes, on wake the socket reconnects → IDENTIFY → READY only refills presence+read_state. **Channels, messages, typing, member presences are all stale and silently wrong.** New messages sent during the gap never arrive — the user sees an empty timeline that disagrees with what others see. This is the single most user-visible bug class waiting to happen.
**How** (pragmatic v1, no server change): on `onopen` after a prior `onclose`, after IDENTIFY/READY completes, also call `fetchChannels()` + `fetchMessages(activeChannelId)` and replace via `setChannels`/`setMessages`. Mark this as a stopgap; track RESUME as proper fix.
**Risk**: Re-fetch storm if many tabs reconnect simultaneously; need a debounce or jitter (already present for backoff). Message list replace can interrupt user scroll position — re-fetch should preserve scroll anchor.
**Test Impact**: New gateway-subscriptions test simulating disconnect→reconnect. WebSocket store test for re-fetch trigger.
**Effort**: M (v1 re-fetch) / L (full RESUME)
**Priority**: **P0** — silent staleness after sleep is a correctness bug, not a perf issue.

---

### 4. Cascade cache invalidation on CHANNEL_DELETE (and add a `MemberStore`)
**What**: When a channel is removed, clear its `messages[channelId]`, `readStates[channelId]`, `unreadChannels[channelId]`, and `typingUsers[channelId]` entries. Also: extract a `useMemberStore` so non-bot members aren't discarded.
**Where**: `stores/useChannelStore.ts` (`removeChannel` becomes a coordinator OR), or wire it in `gateway-subscriptions.ts` `CHANNEL_DELETE` handler to call into multiple stores; new `stores/useMemberStore.ts`.
**Why**: Today `removeChannel` only updates `channels` + `activeChannelId`. The other 3 stores keep entries forever — small memory leak, but more importantly: if the same channel_id is recreated (rare but possible in tests/dev), stale messages reappear. Member side: `useBotStore.fetchBots` calls `/members` and *throws away the humans*. The very next feature (member sidebar, mentions autocomplete, DM) needs them.
**How**:
- Add `clearChannel(channelId)` to MessageStore, ReadStateStore, TypingStore; CHANNEL_DELETE handler calls all four.
- New `useMemberStore`: `members: Record<userId, GuildMember>`; `setMembers`, `addMember`, `removeMember`, `getMember(id)`. `BotStore` becomes a selector on top: `bots = members.filter(m => m.user.bot)`.
**Risk**: Low. Mostly additive.
**Test Impact**: Store tests for cascade. BotStore test updates to use MemberStore.
**Effort**: S (cascade) + S (member store) = S/M
**Priority**: P1

---

### 5. Implement optimistic send with nonce reconciliation
**What**: `sendMessage` inserts a local message with `nonce` and `state: "pending"` immediately; the server echo via `MESSAGE_CREATE` (carrying the same nonce) replaces it; failure flips to `state: "failed"` with retry.
**Where**: `stores/useMessageStore.ts` (`addPendingMessage`, `confirmMessage(nonce, msg)`, `failMessage(nonce)`), `lib/api.ts` (`sendMessage` accepts nonce), `lib/gateway-subscriptions.ts` MESSAGE_CREATE handler (reconcile by nonce), `Message` type (`nonce?`, `state?`).
**Why**: Current send is REST → wait round-trip → store update via Gateway. Visible latency on slow networks; users hit Send twice; no retry surface. This is table stakes for any chat UI and the architecture already supports it (the dispatcher decouples create-source from store update).
**Risk**: Duplicate-display window if MESSAGE_CREATE arrives before REST response sets the nonce (unlikely but real). Mitigated by reconciling on either side — whichever lands second is a no-op.
**Test Impact**: New MessageStore tests (pending/confirm/fail). gateway-subscriptions test for nonce-replace path.
**Effort**: M
**Priority**: P1

---

### 6. Heartbeat ACK tracking + zombie connection detection
**What**: Track `lastHeartbeatAck` timestamp; if next heartbeat tick fires and no op 11 received since prior send, force-close the socket and reconnect.
**Where**: `stores/useWebSocketStore.ts` (op 11 handler currently just `return`s — line ~64; heartbeat interval body).
**Why**: TCP half-open connections survive routing changes (laptop wifi→ethernet, NAT timeout). The browser reports the socket as OPEN but nothing flows. Discord's heartbeat-ACK check is the standard mitigation — without it, Cove relies on `onclose` firing, which can take minutes on a half-open socket. Cheap to add, prevents the "I'm online but no messages arriving" zombie state.
**How**: Module-scoped `lastAck = Date.now()`; on op 11 set `lastAck = Date.now()`; in heartbeat tick, if `Date.now() - lastAck > interval * 1.5`, call `ws.close(4000, "zombied")` → existing onclose path triggers reconnect.
**Risk**: False positives on slow networks; pick threshold conservatively (1.5–2× interval). Don't tear down during initial HELLO window.
**Test Impact**: WebSocket store test with fake timers.
**Effort**: S
**Priority**: P1

---

### 7. Make gateway-dispatcher event registry explicit and warn on unknown events
**What**: Replace the duplicated `gatewayEvents` Set in `useWebSocketStore.ts:90-101` with a single source of truth derived from `GatewayEventMap` keys; warn (dev mode) when an unknown `t` arrives.
**Where**: `lib/gateway-dispatcher.ts` (export `GATEWAY_EVENT_NAMES: ReadonlySet<keyof GatewayEventMap>`), `stores/useWebSocketStore.ts` (consume it).
**Why**: Adding a new event today requires editing two files; forgetting one means events arrive at the WS but never dispatch — silent. The duplication is small but it's a footgun that will bite when GUILD_CREATE, MESSAGE_REACTION_ADD, USER_UPDATE etc. land. Dev-mode warn surfaces server/client schema drift fast.
**How**: Build the set from a typed constant: `export const GATEWAY_EVENT_NAMES = new Set<keyof GatewayEventMap>([...]) satisfies ReadonlySet<keyof GatewayEventMap>`. Better: derive from a values-object so TS catches missing entries.
**Risk**: Trivial.
**Test Impact**: One dispatcher test for the warn behavior.
**Effort**: S
**Priority**: P2

---

## Summary

| # | Title | Priority | Effort |
|---|---|---|---|
| 1 | Guild store + key by guild_id | P1 (P0 if multi-guild near) | L |
| 2 | READY as canonical snapshot | P1 | M |
| 3 | **Reconnect state recovery** | **P0** | M→L |
| 4 | CHANNEL_DELETE cascade + MemberStore | P1 | S/M |
| 5 | Optimistic send with nonce | P1 | M |
| 6 | Heartbeat ACK / zombie detection | P1 | S |
| 7 | Single source of truth for event names | P2 | S |

**Recommended order**: 3 (correctness) → 6 (cheap, prevents related bug class) → 4 (foundation for member features) → 1+2 together (the architectural pivot) → 5 (UX polish) → 7 (cleanup along the way).

**One-sentence verdict**: The current code is competent for a single-guild MVP, but three load-bearing assumptions — *single guild, READY is partial, reconnect just re-opens the socket* — need to be retired before the next feature wave, and #3 is already a user-visible correctness bug on laptop sleep.
