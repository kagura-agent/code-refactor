# Cove Client State Management Architecture Audit — 2026-06-05

Scope reviewed:
- `packages/client/src/stores/*.ts` — WebSocket, Channel, Message, User, Presence, ReadState, Typing, Bot, Theme stores
- `packages/client/src/lib/*.ts` — API wrapper, gateway dispatcher/subscriptions, markdown/theme/avatar helpers and tests
- `packages/client/src/types.ts`
- Supporting read for behavior confirmation: `App.tsx`, `MessageList.tsx`, `MessageInput.tsx`, `Sidebar.tsx`, `MemberList.tsx`, and server/shared gateway type snippets

Comparison target: Discord-style client cache: `guild -> channel -> message/member/presence/read-state`, READY seeds cache/session state, gateway events incrementally mutate cache, reconnect either resumes missed events or invalidates/refetches affected state, optimistic UI uses pending entities reconciled by ACK/events, multi-guild identity scopes every cache key and event.

## Current State Summary

Cove has a clean small-app split, but it is not yet a Discord-like cache. The client currently behaves like a single-guild chat app with scattered per-feature state:

- Channels are flat: `ChannelStore.channels: Channel[]`, `activeChannelId: string | null`.
- Messages are only grouped by channel: `MessageStore.messages: Record<channelId, Message[]>`.
- No `GuildStore`; `api.getGuildId()` fetches `/users/@me/guilds` once and caches `guilds[0].id` globally.
- No `MemberStore`; members are fetched ad hoc by `MemberList` and `BotStore.fetchBots()` via `fetchMembers()`.
- Presence is a global `Set<userId>`, not scoped to guild/session visibility.
- Read state is keyed only by channel, which works while channel IDs are global, but it has no guild/session lifecycle integration.
- READY only initializes presences and read states. It ignores `user`, `guilds`, session metadata, and does not populate channels/members/messages.
- Gateway client supports HELLO/IDENTIFY/HEARTBEAT/ACK and reconnect backoff, but does not track sequence/session/resume and does not re-fetch caches after reconnect.
- Gateway dispatcher/client allowlist lacks events that the server can already send (`GUILD_CREATE`, `GUILD_DELETE`, `MESSAGE_DELETE_BULK`). Those events are currently dropped in `useWebSocketStore` before subscriptions can see them.
- Mutations are mostly server-confirmed, not optimistic. `MessageInput` clears input but does not add a pending local message; `Sidebar` waits for create/delete response and also applies local mutations while gateway events may also arrive.

This is acceptable for the current single-guild MVP, but it will create correctness debt as soon as multi-guild, reconnect reliability, member lists, or richer gateway events matter.

---

### [Introduce a Guild-Scoped Entity Cache]
**What**: Replace flat feature lists with a normalized cache rooted at guilds, with explicit guild selection and entity lookup indexes.

**Where**: `packages/client/src/stores/useChannelStore.ts`, `useMessageStore.ts`, new `useGuildStore.ts`, likely new selectors/hooks; `packages/client/src/lib/api.ts`; callers in `App.tsx`, `Sidebar.tsx`, `MessageList.tsx`, `MemberList.tsx`.

**Why**: Discord clients are guild-centric. Today Cove has no guild store and implicitly chooses `guilds[0]` in the API layer. That makes multi-guild hard because channel lists, active channel, read state, presence visibility, member lists, and cache invalidation all lack a parent scope. It also hides a product decision inside `api.ts` rather than state/UI.

**How** (high-level):
- Add a `GuildStore` that owns:
  - `guildsById: Record<string, Guild>`
  - `guildIds: string[]`
  - `activeGuildId: string | null`
  - optional `guildLoadState/guildVersion` metadata
- Refactor `ChannelStore` from `channels: Channel[]` to either:
  - normalized `channelsById + channelIdsByGuildId`, or
  - minimally `channelsByGuildId: Record<guildId, Channel[]>`.
- Scope active channel by guild: `activeChannelIdByGuildId` or `activeChannelId` derived from active guild.
- Stop `api.getGuildId()` from silently picking the first guild for UI operations. API calls should accept `guildId` when they are guild-scoped; selection belongs in `GuildStore`/UI.
- Keep messages keyed by channel ID, but connect them to guild lifecycle via `channel.guild_id`; on guild removal, clear child channel/message/member/read/typing state.

**Risk**: Medium migration risk because many components assume a flat current guild. Selectors must be careful to avoid excessive React re-renders. There is also a temporary awkward phase if the server READY payload contains guilds but not channels.

**Test Impact**: Existing tests that mock channel state will break. Need new store tests for active guild/channel selection, channel add/update/delete within guild, and guild delete cascading cleanup. Component tests/mocks using `useChannelStore` need updates.

**Effort**: L

**Priority**: P0

---

### [Handle the Full Gateway Event Surface]
**What**: Make the client gateway event map and subscriptions match events the server can emit, especially guild and bulk-delete events.

**Where**: `packages/client/src/lib/gateway-dispatcher.ts`, `gateway-subscriptions.ts`, `packages/client/src/stores/useWebSocketStore.ts`, channel/message/guild stores, tests in `gateway-dispatcher.test.ts` and `gateway-subscriptions.test.ts`.

**Why**: `useWebSocketStore` drops any dispatch event not in its `gatewayEvents` allowlist. The server currently emits `GUILD_CREATE`, `GUILD_DELETE`, and `MESSAGE_DELETE_BULK`, but the client does not type or subscribe to them. That means membership changes and bulk clears can leave stale UI/cache. This is especially important because recent server work added guild membership events.

**How** (high-level):
- Extend `GatewayEventMap` with:
  - `GUILD_CREATE: Guild`
  - `GUILD_DELETE: { id: string }`
  - `MESSAGE_DELETE_BULK: { ids: string[]; channel_id: string; guild_id: string }`
  - READY fields already sent by server: `user`, `guilds`, `session_id`, `v`.
- Add these event names to `gatewayEvents` allowlist, or better derive the allowlist from typed subscriptions/dispatcher metadata to avoid future drift.
- Subscribe handlers:
  - `GUILD_CREATE`: upsert guild, optionally fetch its channels/members or mark guild cache stale.
  - `GUILD_DELETE`: remove guild and cascade clear channels/messages/typing/read/member state for that guild.
  - `MESSAGE_DELETE_BULK`: remove all listed messages from the channel and reconcile channel `last_message_id` if present/needed.
- Make `CHANNEL_DELETE` also clear child message/typing/read state, not just remove the channel row.

**Risk**: Low-to-medium. Event typing is straightforward, but cascade cleanup can accidentally clear state for channels that are still reachable if future channel IDs are not globally unique or if delete events are duplicated/out of order.

**Test Impact**: Add gateway subscription tests for each missing event and for ignored duplicate/out-of-order cases. Existing tests should mostly extend, not fail, except mocks that omit new store methods.

**Effort**: M

**Priority**: P0

---

### [Use READY as the Initial Cache Seed]
**What**: Treat READY as session initialization for user, guilds, presences, read states, and cache freshness, not just presences/read states.

**Where**: `packages/client/src/lib/gateway-dispatcher.ts`, `gateway-subscriptions.ts`, `stores/useUserStore.ts`, new `useGuildStore.ts`, `useChannelStore.ts`, `packages/client/src/App.tsx`, and possibly server READY payload if channels/members are added there later.

**Why**: Discord clients hydrate their known session/guild state from READY and then use incremental events. Cove currently performs REST fetches in `App.tsx` (`fetchMe`, `fetchChannels`) and separately processes READY only for `presences`/`read_state`. This causes split-brain initialization: the websocket has authoritative session data (`user`, `guilds`, `session_id`) but the client ignores most of it, while API initialization hardcodes the first guild. It also makes reconnect recovery harder because there is no single “session is ready, caches are valid/stale” transition.

**How** (high-level):
- Update READY type to include the actual server payload: `{ v, user, guilds, session_id, presences, read_state }`.
- On READY:
  - set current user from `data.user` if not already set;
  - seed `GuildStore` with `data.guilds`;
  - choose/preserve active guild;
  - initialize presences/read states;
  - mark guild/channel/member/message caches as stale or ready according to what READY contains.
- Move initial `fetchChannels()` out of blind app startup and make it depend on active guild after READY/guild selection.
- Consider server-side READY expansion later: include channels and lightweight member/presence summaries per guild, like Discord does, or explicitly document that READY only seeds guild/session and client must fetch channels per guild.

**Risk**: Medium. Changing startup ordering can introduce race conditions where components render before active guild/channel is chosen. Need a clear state machine: auth pending -> gateway identifying -> ready -> guild/channel loading.

**Test Impact**: App startup tests and gateway subscription tests need updates. Add tests proving READY with multiple guilds does not overwrite a user-selected active guild unnecessarily.

**Effort**: M/L

**Priority**: P1

---

### [Reconnect Recovery and Cache Invalidation]
**What**: Add explicit reconnect recovery: either real Gateway RESUME with sequence/session tracking, or a conservative REST revalidation path after reconnect.

**Where**: `packages/client/src/stores/useWebSocketStore.ts`, `packages/client/src/lib/gateway-dispatcher.ts`, `gateway-subscriptions.ts`, `api.ts`, channel/message/read/member stores, potentially server gateway support for resume/replay.

**Why**: The current websocket store reconnects with exponential backoff, but state recovery is blind. If the socket is down while messages/channels/read states change, the local cache never learns. Discord solves this by tracking sequence numbers and resuming sessions; when resume fails, it invalidates and rehydrates from READY/REST. Cove has shared `GatewayOpcode.RESUME` in types, but the client does not store `s`, `session_id`, or send resume.

**How** (high-level):
- Short-term P1 path: on reconnect + READY, mark caches stale and re-fetch:
  - guild list;
  - channels for active/known guilds;
  - latest messages for active channel and channels with unread/last-message changes;
  - read states and members as needed.
- Long-term Discord-like path:
  - store last dispatch sequence `s` from every payload;
  - store `session_id` from READY;
  - send `RESUME` on reconnect with token/session/seq;
  - server replays missed events or returns invalid-session forcing full rehydrate.
- Add a `connectionEpoch`/`cacheEpoch` in state so components can refetch only when recovery completes or cache is marked stale.
- Clear transient typing state on disconnect (already done) but keep durable messages/channels until revalidated to avoid jarring UI.

**Risk**: Real resume requires server-side event buffers and invalid-session semantics; implementing only client resume without server replay gives false confidence. REST revalidation is easier but may be heavier and can still miss historical gaps if only the last 50 messages are fetched.

**Test Impact**: Need websocket-store tests around reconnect transitions and gateway payload sequence handling. Need integration tests simulating events during disconnect and verifying refetch/reconciliation.

**Effort**: M for REST revalidation, L for full RESUME/replay.

**Priority**: P0 for some recovery path; P2 for full Discord-grade resume if not needed immediately.

---

### [Add Member Store and Scope Presence to Guild Membership]
**What**: Promote guild members from ad-hoc component fetches into a real cache, and make presence/read derivations operate on guild-scoped member visibility.

**Where**: New `packages/client/src/stores/useMemberStore.ts`, `usePresenceStore.ts`, `useBotStore.ts`, `MemberList.tsx`, `BotManagement.tsx`, `api.ts`, `gateway-subscriptions.ts`.

**Why**: Discord separates user, guild member, and presence caches. Cove currently has global `onlineUsers: Set<userId>` and fetches members ad hoc in `MemberList`, while `BotStore` fetches all members just to filter bots. This duplicates network calls, blocks coherent multi-guild UI, and makes presence look global rather than “online among users sharing this guild”. Bot creation/deletion also mutates only `BotStore`, not a canonical member cache.

**How** (high-level):
- Add `MemberStore`:
  - `membersByGuildId: Record<guildId, Record<userId, GuildMember>>` or normalized equivalent;
  - `fetchMembers(guildId)`, `setMembers`, `upsertMember`, `removeMember`, loading/stale flags.
- Refactor `MemberList` to select members for active guild from store.
- Refactor `BotStore` into either a derived selector over members (`user.bot`) or a thin bot-management action layer that updates the member cache.
- Keep `PresenceStore` as user-level presence (`presenceByUserId`) but selectors should join presence with current guild members. If future per-guild invisible/role semantics arrive, presence selectors can incorporate guild membership.
- Add gateway member events when server supports them; until then, mark member cache stale after bot create/delete and refetch.

**Risk**: Member caching may expose stale member lists until server emits member add/update/delete events. Derived bot state needs careful migration so bot-management screens keep working.

**Test Impact**: `BotStore` tests/mocks will change. Add member-store fetch/cache tests and component tests for online/offline grouping from cached members plus presence store.

**Effort**: M

**Priority**: P1

---

### [Make Mutations Optimistic but Reconciled]
**What**: Add optimistic entities/actions for send message, create/delete channel, clear messages, and read acknowledgements, with server/gateway reconciliation.

**Where**: `packages/client/src/stores/useMessageStore.ts`, `useChannelStore.ts`, `useReadStateStore.ts`, `gateway-subscriptions.ts`, `MessageInput.tsx`, `Sidebar.tsx`, `ChatArea.tsx`, `api.ts`.

**Why**: The current UX is partly optimistic at the input level (`MessageInput` clears text before POST completes), but cache updates are server-confirmed. That means sent messages appear only after REST response or gateway echo, and failed sends restore input but no pending/error message exists. Channel create/delete currently updates after REST and may also receive gateway echo; add is idempotent but delete can produce confusing state if REST succeeds and gateway is delayed/duplicated. Discord-like clients show pending local state immediately and reconcile with authoritative gateway events.

**How** (high-level):
- Extend message state with pending/error metadata or a separate `localMessages` overlay:
  - create temporary message ID on send;
  - render pending state immediately;
  - replace temp with server message from POST or `MESSAGE_CREATE` echo;
  - mark failed send with retry/delete options rather than only restoring text.
- For read acks, optimistically clear unread on active-channel view but reconcile on `MESSAGE_ACK`; keep a failed/dirty flag if ack fails.
- For channel create/delete, use operation IDs or pending flags:
  - pending channel row until `CHANNEL_CREATE` confirms;
  - tombstone deleted channel to prevent gateway lag from resurrecting it incorrectly;
  - clear child caches when delete is confirmed.
- Ensure all optimistic mutations are scoped by guild/channel and idempotent with gateway event handlers.

**Risk**: Optimistic reconciliation is easy to get subtly wrong, especially if REST returns and gateway echo arrive in different orders. Temporary IDs must not leak into read-state/ack APIs. Pending delete tombstones need expiration or authoritative refetch to avoid permanent ghosts.

**Test Impact**: Existing UI assumptions may change. Need store-level tests for POST success, POST failure, REST-before-gateway, gateway-before-REST, duplicate events, and retry behavior.

**Effort**: M/L

**Priority**: P2

---

## Cross-Cutting Notes

- `api.resetGuildId()` exists but `useUserStore.logout()` does not call it. Even before multi-guild, logout/login as a different user can retain a stale cached guild ID in module memory. This should be fixed as part of the guild-store/API refactor, or as a small safety patch sooner.
- `CHANNEL_DELETE` currently removes the channel but does not clear `MessageStore.messages[channelId]`, `TypingStore.typingUsers[channelId]`, or read state. Hidden stale entries will accumulate and may reappear if IDs collide or if UI references a removed active channel.
- `MessageStore.setMessages(channelId, messages)` overwrites a channel wholesale. That is simple but can race with gateway `MESSAGE_CREATE` while a fetch is in flight: fetch older snapshot lands after a newer gateway message and drops it. Use per-channel version/request IDs or merge-by-id on fetch results.
- `PresenceStore.initPresences()` replaces the entire global online set. For multi-guild reconnect/READY, this is only correct if READY includes all users visible across all current guilds. It currently does server-side shared-guild filtering; OK for now, but client selectors should not assume global presence means global roster membership.
- `GatewayDispatcher` has typed events but no runtime validation. Given the server and client live in one monorepo, importing shared gateway event types or generating client event map from shared types would reduce drift.

## Recommended Sequencing

1. P0: Add missing gateway event handling + cascade cleanup (`GUILD_CREATE`, `GUILD_DELETE`, `MESSAGE_DELETE_BULK`, `CHANNEL_DELETE` child cleanup).
2. P0: Add reconnect cache recovery through conservative REST revalidation after reconnect/READY.
3. P0/P1: Introduce `GuildStore` and remove implicit `guilds[0]` API cache from UI flows.
4. P1: Treat READY as session/guild initialization and make startup state machine explicit.
5. P1: Add `MemberStore`; make bot/member/presence views derive from it.
6. P2: Add optimistic mutation layer once cache identity and event reconciliation are stable.

The architectural direction I recommend is not “one giant Zustand store”; keep stores modular, but make them share a normalized Discord-like identity model: guild IDs at the top, channel/member/read/typing/message state scoped below, and gateway events as the only incremental mutation surface after initial REST/READY hydration.
