# Cove Client State Management Refactoring Proposals

## [Multi-Guild Hierarchy Migration]
**What**: Restructure stores to natively support the Guild -> Channel -> Message hierarchy.
**Where**: `useChannelStore.ts`, `useMessageStore.ts`, `api.ts`
**Why**: Currently, `api.ts` hardcodes a single cached `_guildId` and channels are stored as a flat array. This fundamentally blocks future multi-guild support and deviates from Discord's client cache topology (where everything is scoped by guild).
**How**: Create a new `useGuildStore.ts`. Refactor `useChannelStore` to use `Record<string, Channel[]>` (keyed by guild ID), and update `api.ts` to stop caching a singleton guild ID. Update UI selectors to accept a `guildId` context.
**Risk**: Breaking changes across all navigation and chat UI components that expect a flat channel array.
**Test Impact**: High. Will break any tests assuming `useChannelStore.channels` is a flat array.
**Effort**: L
**Priority**: P0

## [READY Payload Cache Hydration]
**What**: Populate core caches (Channels, Guilds, CurrentUser) from the `READY` event payload.
**Where**: `gateway-subscriptions.ts`, `useChannelStore.ts`, `useUserStore.ts`
**Why**: Discord's model hydrates the entire core client state (guilds, channels, settings, members) during the `READY` event. Cove currently only initializes presences and read states on `READY`, requiring scattered, ad-hoc REST calls to populate channels and user data.
**How**: Expand the `READY` event handler in `gateway-subscriptions.ts` to dispatch `setChannels` and `setGuilds`. The backend will need to include these in the READY payload.
**Risk**: Increased initial WebSocket payload size.
**Test Impact**: Minor. Mostly requires updating `gateway-subscriptions.test.ts` to mock the new payload fields.
**Effort**: M
**Priority**: P1

## [WebSocket Reconnect State Recovery]
**What**: Implement state resynchronization after WebSocket reconnection.
**Where**: `useWebSocketStore.ts`, `gateway-subscriptions.ts`, `api.ts`
**Why**: The client implements exponential backoff for reconnects, but suffers from "reconnect blindness". When it reconnects, it simply clears typing states but fails to fetch any `MESSAGE_CREATE` or `CHANNEL_UPDATE` events that occurred during the downtime.
**How**: Implement sequence numbers (`seq`) for Gateway events. On reconnect, send a `RESUME` payload instead of `IDENTIFY`. If the session is invalid, re-fetch the active channel's message history and the guild's channel list via REST.
**Risk**: Race conditions between REST data fetching and incoming live Gateway events immediately following a reconnect.
**Test Impact**: Will require new integration tests for reconnect/resume lifecycle.
**Effort**: M
**Priority**: P0

## [Optimistic UI for Message Mutations]
**What**: Immediately reflect message creation/edits in the UI before server acknowledgement.
**Where**: `useMessageStore.ts`, components dispatching API calls.
**Why**: Currently, the UI feels sluggish because it waits for a round-trip REST response to render a sent message. Discord relies heavily on optimistic updates with unique nonces.
**How**: Extend the local `Message` type with `pending: boolean` and `nonce: string`. When sending, inject the message into `useMessageStore` immediately. When the Gateway emits `MESSAGE_CREATE`, match the `nonce`, remove the pending flag, and overwrite the temporary ID with the real snowflake ID.
**Risk**: Risk of duplicated messages in the UI if nonce reconciliation fails or if the Gateway event arrives before the optimistic insert.
**Test Impact**: High. Message creation tests will need to assert the immediate state change and the subsequent reconciliation.
**Effort**: M
**Priority**: P1

## [Unified Member/User Caching]
**What**: Consolidate `BotStore` and `UserStore` into a normalized `MemberStore`.
**Where**: `useBotStore.ts`, `useUserStore.ts`, `usePresenceStore.ts`
**Why**: Cove treats users and bots differently (fetching bots ad-hoc in `useBotStore`), which fragments the identity cache. In Discord, bots are just users with a `bot: true` flag, wrapped in a `GuildMember` object scoped to a guild.
**How**: Deprecate `useBotStore`. Create `useMemberStore` keyed by `guildId` and `userId`. Normalize user objects to a global `UserStore`, and keep guild-specific metadata (nicknames, roles) in the `MemberStore`.
**Risk**: Refactoring data flow for bots might temporarily break bot interaction UI.
**Test Impact**: Moderate. Tests mocking `useBotStore` will need migration to the new member store.
**Effort**: M
**Priority**: P2
