# Cove API & Gateway Refactoring Analysis (Discord v10)

### 1. Add `GET /gateway/bot` Endpoint
**What**: Implement `GET /gateway/bot` alongside the existing `/gateway` endpoint.
**Where**: `server/src/app.ts`
**Why**: Discord bot libraries (discord.js, discordgo, etc.) strictly rely on `/gateway/bot` to fetch the WebSocket URL and sharding metadata. A 404 here will prevent standard bots from connecting entirely.
**How**: Add `app.get(`${API_PREFIX}/gateway/bot`, ...)` returning `{ url: gwUrl, shards: 1, session_start_limit: { total: 1000, remaining: 1000, reset_after: 0, max_concurrency: 1 } }`.
**Risk**: Low.
**Test Impact**: None (pure addition).
**Effort**: S
**Priority**: P0

### 2. Standardize Discord `Message` Payload Shape
**What**: Add missing mandatory array fields to the `Message` object.
**Where**: `@cove/shared/src/types.ts`, `server/src/repos/messages.ts`
**Why**: Strictly typed Discord clients expect `Message` objects to contain certain arrays (even if empty). Omitting them causes deserialization errors or crashes.
**How**: Update the `Message` interface and `toMessage` builder to include `attachments: []`, `embeds: []`, `mentions: []`, `mention_roles: []`, `pinned: false`, `mention_everyone: false`, and `tts: false`.
**Risk**: Low.
**Test Impact**: Tests asserting strict message shape will need to be updated.
**Effort**: S
**Priority**: P0

### 3. Replace Wildcard DELETE with Discord Bulk Delete
**What**: Remove `DELETE /channels/:id/messages` and implement `POST /channels/:id/messages/bulk-delete`.
**Where**: `server/src/routes/messages.ts`, `server/src/repos/messages.ts`
**Why**: The current endpoint deletes *all* messages in a channel, which is non-standard and highly dangerous. Discord API expects a `POST` endpoint accepting an array of specific message IDs to delete.
**How**: Remove `app.delete("/channels/:id/messages")`. Add `app.post("/channels/:id/messages/bulk-delete")` expecting `{ messages: string[] }`. Update the repository to delete by ID array and dispatch `MESSAGE_DELETE_BULK`.
**Risk**: Moderate, fundamentally changes how bulk deletion works.
**Test Impact**: Bulk delete tests will fail and must be rewritten.
**Effort**: M
**Priority**: P0

### 4. Remove custom Gateway `REQUEST_TYPING` Opcode
**What**: Remove Opcode 4 mapping for typing; rely strictly on REST typing.
**Where**: `@cove/shared/src/types.ts`, `server/src/ws/index.ts`
**Why**: Discord API does not use Gateway messages to send typing events from client to server (Opcode 4 is actually `VOICE_STATE_UPDATE`). Typing is correctly triggered via REST `POST /channels/{channel.id}/typing`, which Cove already implements.
**How**: Remove `REQUEST_TYPING = 4` from `GatewayOpcode`. Drop the case in `ws/index.ts`.
**Risk**: Low, but requires clients using the non-standard opcode to switch to the REST endpoint.
**Test Impact**: Tests asserting Gateway typing will need to use the HTTP route.
**Effort**: S
**Priority**: P1

### 5. Align `READY` Payload and implement `GUILD_CREATE` lazy loading
**What**: Change `READY` payload to send unavailable guilds and dispatch `GUILD_CREATE` afterward.
**Where**: `server/src/ws/session.ts`, `server/src/ws/dispatcher.ts`
**Why**: Discord v10 spec requires `READY` to contain an array of guilds marked as `{ id, unavailable: true }` and include `resume_gateway_url`. Full guild data must follow via individual `GUILD_CREATE` events. Some clients will incorrectly cache or ignore guilds if sent fully populated in `READY`.
**How**: In `session.ts`, map `guilds` to `{ id, unavailable: true }` and include `resume_gateway_url`. Immediately after sending `READY`, loop through the guilds and dispatch `GUILD_CREATE` events to the session.
**Risk**: Moderate, changes the initial gateway connection flow.
**Test Impact**: Tests checking the `READY` payload for full guild data will break.
**Effort**: M
**Priority**: P1

### 6. Handle `RESUME` Gateway Opcode Gracefully
**What**: Respond to Gateway Opcode 6 (`RESUME`) with Opcode 9 (`INVALID_SESSION`).
**Where**: `server/src/ws/index.ts`
**Why**: Cove doesn't currently support session resumption. When Discord clients reconnect, they attempt to resume by sending Opcode 6. Ignoring it causes clients to hang indefinitely instead of falling back to a full `IDENTIFY`.
**How**: Add `case GatewayOpcode.RESUME:` block that sends Opcode 9 with `d: false` (indicating the session cannot be resumed and they must re-identify).
**Risk**: Low.
**Test Impact**: None, improves client compatibility and stability.
**Effort**: S
**Priority**: P1

### 7. Refactor Error Responses to match Discord `errors` Object
**What**: Update validation errors to return the standard Discord `errors` tree alongside `code` and `message`.
**Where**: `server/src/validation.ts`
**Why**: Discord API v10 uses a detailed `errors` tree for code 50035 (Invalid Form Body). Strictly typed clients parsing HTTP 400 responses expect this shape to map validation errors to specific fields.
**How**: Refactor `validationError` to construct the `{ code: 50035, message, errors: { [field]: { _errors: [{ code: "...", message: "..." }] } } }` structure instead of returning just the message and code.
**Risk**: Low.
**Test Impact**: Tests checking validation error responses will need to assert the new nested structure.
**Effort**: M
**Priority**: P2
