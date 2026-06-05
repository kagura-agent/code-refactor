# Cove OpenClaw Plugin Refactoring Analysis

## 1. Gateway RESUME & Sequence Tracking
**What**: Implement Discord-compatible `RESUME` flow (Opcode 6) with sequence tracking.
**Where**: `gateway-client.ts`
**Why**: Addresses #224. Currently, reconnects drop state and perform a full `IDENTIFY`. This leads to missed events during connection drops (no `seq` tracking, no `session_id`).
**How**: 
- Track `seq` (`s`) from all incoming Gateway payloads.
- Store `session_id` from the `READY` payload.
- On disconnect, if `session_id` and `seq` exist, send `RESUME` (Opcode 6) instead of `IDENTIFY`.
- Handle `INVALID_SESSION` (Opcode 9) by falling back to `IDENTIFY`.
**Risk**: Infinite resume loops if Opcode 9 isn't handled correctly.
**Test Impact**: Requires new tests mocking gateway connection drops and asserting `RESUME` payload format.
**Effort**: M
**Priority**: P0

## 2. Multi-Guild Support & Dynamic GuildStore
**What**: Remove hardcoded `"cove"` fallback and parse `GUILD_CREATE`.
**Where**: `channel.ts`, `types.ts`, `rest-client.ts`
**Why**: Addresses #228. `guildId` defaults to `"cove"` in `rest-client.ts` (`getChannels`) and `channel.ts` config reading. Blocks expanding the bot to multiple guilds.
**How**: 
- Remove `guildId = "cove"` defaults.
- Add `GUILD_CREATE` event handling to `gateway-client.ts` to build a dynamic list of active guilds.
- Pass explicitly configured or dynamically discovered guild IDs to REST calls.
**Risk**: Potential breakage for setups that omitted `guildId` from config assuming the `"cove"` default.
**Test Impact**: Need to update existing tests to explicitly pass mock guild IDs.
**Effort**: S
**Priority**: P1

## 3. REST Client Reliability (Rate Limits)
**What**: Add fetch retries for 429 Rate Limits and 5xx errors.
**Where**: `rest-client.ts`
**Why**: Addresses REST client error handling. Currently, `fetch` throws immediately on non-ok status. High volume of tool progress updates can hit rate limits and break dispatches.
**How**: Wrap `fetch` calls in a retry handler that reads the `Retry-After` header on 429s, sleeps, and retries. Add exponential backoff for 5xx responses (up to 3 retries).
**Risk**: Queue build-up if `Retry-After` is long; possible timeout on OpenClaw's side.
**Test Impact**: Needs mocked HTTP 429 responses to assert retry delays.
**Effort**: M
**Priority**: P1

## 4. Reconnect State Recovery Sync
**What**: Re-sync channel histories when a Gateway connection fully resets.
**Where**: `channel.ts`, `gateway-client.ts`
**Why**: Addresses #229. If a `RESUME` fails and the bot re-identifies, events during the downtime are permanently lost. Active dispatches could stall or be based on stale state.
**How**: Distinguish between "resume successful" and "full reconnect" in Gateway events. On a full reconnect, `channel.ts` should re-fetch `getMessages` via REST for active channels to reconcile missed updates before resuming message processing.
**Risk**: Race conditions between fetching historical messages and receiving new ones from the Gateway.
**Test Impact**: High. Requires testing the full reconnection pipeline with missed messages.
**Effort**: L
**Priority**: P1

## 5. Gateway Opcode 4 Collision Cleanup
**What**: Restrict Gateway `send()` usage to prevent `REQUEST_TYPING` abuse.
**Where**: `gateway-client.ts`, `channel.ts`
**Why**: Addresses #214. The `send(payload)` method in `gateway-client.ts` is public with the comment `/** Public: used by channel handler to send typing indicators. */`. Discord uses Opcode 4 for `VOICE_STATE_UPDATE`. Sending typing over WS violates Discord's spec (typing is REST).
**How**: Change `send()` to `private` or `protected`. Ensure `channel.ts` strictly uses `restClient.sendTyping()` and remove the misleading comment.
**Risk**: None. `channel.ts` currently uses REST anyway, so this just removes an invalid API surface.
**Test Impact**: Minimal typing updates in tests if any mock called `gateway.send`.
**Effort**: S
**Priority**: P2
