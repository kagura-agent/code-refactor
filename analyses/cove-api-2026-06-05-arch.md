# Cove REST API + WebSocket Gateway Discord v10 Compatibility Audit — Arch

Scope audited: `packages/server/src/routes/*.ts`, `packages/server/src/ws/*.ts`, `packages/server/src/app.ts`, `packages/server/src/auth.ts`, `packages/server/src/validation.ts`, and `packages/shared/src/types.ts` in `~/repos/forks/cove`.

External reference check: fetched Discord developer docs on 2026-06-05 with `https_proxy=http://127.0.0.1:1083`; direct docs pages are JS-heavy but confirmed key v10 reference points from rendered text: Gateway opcode table, channel endpoints, guild/user endpoint lists, and common JSON error codes.

## Overall read

Cove is intentionally Discord-shaped, but it is currently closer to a custom chat API with Discord-like names than a drop-in Discord API v10 surface. The most important compatibility gaps are:

- Gateway opcode `4` is used for a Cove typing extension, but Discord opcode `4` is `Voice State Update`; typing is a REST endpoint/event, not a Gateway client opcode.
- Major Discord Gateway lifecycle pieces are absent (`RESUME`, `RECONNECT`, `INVALID_SESSION`, identify `intents`/`properties`, server-initiated heartbeat handling semantics).
- Object serializers are minimal and omit many required/commonly expected Discord fields.
- REST error envelopes and permission/error semantics are inconsistent and mostly ad hoc.
- Authorization is global “any authenticated user can mutate many global resources,” not Discord’s permission model.
- Some core endpoints/events expected by Discord clients are absent or return non-Discord shapes.

Below are independent refactoring opportunities, not a request to rewrite everything.

---

### Fix Gateway Opcode Contract and Handshake Semantics

**What**: Align Gateway opcodes and identify/heartbeat semantics with Discord v10; move Cove-only typing request off opcode `4`.

**Where**: `packages/shared/src/types.ts`, `packages/server/src/ws/index.ts`, `packages/server/src/ws/session.ts`, `packages/plugin/src/gateway-client.ts`, any client code sending `GatewayOpcode.REQUEST_TYPING`.

**Why**: Cove currently defines `REQUEST_TYPING = 4`, but Discord Gateway opcode `4` is `Voice State Update`. A Discord-compatible client that sends opcode `4` would be interpreted as Cove typing, and a Cove client using opcode `4` is not Discord-compatible. The server also accepts `IDENTIFY` with only `{ token }`, while Discord clients normally send `{ token, properties, intents, presence?, compress?, large_threshold?, shard? }`; heartbeat payload `d` should carry the last sequence number, and Discord also allows server-initiated heartbeat requests.

**How** (high-level):
- Replace `REQUEST_TYPING = 4` with Discord’s opcode names/values: `PRESENCE_UPDATE = 3`, `VOICE_STATE_UPDATE = 4`, `RESUME = 6`, `RECONNECT = 7`, `REQUEST_GUILD_MEMBERS = 8`, `INVALID_SESSION = 9`, `HELLO = 10`, `HEARTBEAT_ACK = 11`.
- Treat typing as REST-only (`POST /channels/:id/typing`) for Discord compatibility; if Cove needs Gateway typing as an extension, use a namespaced extension opcode outside Discord’s reserved range and gate it behind a Cove client capability.
- Parse but initially tolerate `IDENTIFY` fields (`properties`, `intents`, `presence`, `shard`) instead of rejecting Discord clients that include them.
- Track the last sequence from each session and accept heartbeat `d` as `number | null`; optionally send opcode `1` heartbeat requests if needed.
- Return `INVALID_SESSION` or close with the appropriate Gateway close code for invalid session state instead of silently ignoring many malformed client events.

**Risk**: Existing Cove plugin/client code may depend on `REQUEST_TYPING = 4`; it will need a coordinated client update or compatibility shim. If strict identify validation is introduced too early, existing minimal Cove clients could break.

**Test Impact**: Gateway tests will need updates for the opcode enum and at least new tests for `IDENTIFY` with Discord-shaped payload, heartbeat `d`, and ensuring opcode `4` is no longer typing. Plugin tests may need updates if they send Gateway typing.

**Effort**: M

**Priority**: P0

---

### Add Discord Gateway Discovery and Session Lifecycle Endpoints

**What**: Make REST Gateway discovery match Discord expectations, including `/gateway/bot`, and ensure returned URLs are usable by Discord-style clients.

**Where**: `packages/server/src/app.ts`, possibly config in `packages/server/src/index.ts`, tests in `packages/server/src/__tests__/api.test.ts`.

**Why**: Cove exposes `GET /api/v10/gateway` returning `{ url }`, which is correct in spirit, but Discord-compatible bot libraries also commonly call `GET /api/v10/gateway/bot` to get `{ url, shards, session_start_limit }`. Returned Gateway URLs should be compatible with clients that append `?v=10&encoding=json`; the WebSocket server currently listens at `/gateway` and does not explicitly account for version/encoding query params or compression negotiation.

**How** (high-level):
- Keep `GET /api/v10/gateway` returning `{ url }`.
- Add `GET /api/v10/gateway/bot` returning at least:
  - `url`
  - `shards: 1`
  - `session_start_limit: { total, remaining, reset_after, max_concurrency }`
- Normalize configured `gatewayUrl` to a base WebSocket URL that can accept query strings; document whether Cove ignores `v`, `encoding`, and `compress` initially.
- Add tests for both endpoints and for WebSocket connection with `/gateway?v=10&encoding=json`.

**Risk**: Low for current clients; mostly additive. The only risk is exposing misleading rate/session fields if clients depend on them. Use conservative fixed values until real rate limiting exists.

**Test Impact**: Additive API tests; no existing tests should need to break unless endpoint auth/public behavior is changed.

**Effort**: S

**Priority**: P1

---

### Introduce Central Discord Serializers for User, Channel, Message, Guild, Member, and Gateway Payloads

**What**: Centralize Discord object response serialization and fill required/commonly expected v10 fields, while keeping Cove extensions explicit.

**Where**: `packages/shared/src/types.ts`, `packages/server/src/repos/*.ts`, `packages/server/src/routes/*.ts`, `packages/server/src/ws/session.ts`, `packages/server/src/ws/dispatcher.ts`.

**Why**: Current shared types are too minimal for many Discord-compatible clients:
- `User` lacks `avatar`, `discriminator`/modern username fields, and uses a minimal shape inconsistently with `CoveAgent`.
- `Message` lacks common fields such as `guild_id`, `mentions`, `mention_roles`, `attachments`, `embeds`, `pinned`, `tts`, `flags`, `components`, etc.
- `Channel` lacks fields like `permission_overwrites`, `nsfw`, `rate_limit_per_user`, `parent_id`, `last_message_id`, and type-specific fields.
- `Guild` in `READY`/`GUILD_CREATE` is only `{ id, name, icon, owner_id }`, whereas Discord Gateway clients often expect guild payloads with at least channels/members/presences or explicit partial/unavailable semantics.

Without central serializers, individual routes and events will drift in shape and it will be hard to know what compatibility level Cove actually supports.

**How** (high-level):
- Create a server-side serialization layer, e.g. `discord/serializers.ts`, with functions like `toDiscordUser`, `toDiscordChannel`, `toDiscordMessage`, `toDiscordGuild`, `toDiscordGuildMember`, `toDiscordReady`.
- Populate unsupported-but-expected fields with safe defaults (`[]`, `null`, `false`, `0`) where Discord permits them, and clearly namespace Cove-only fields (for example `bio`, `read_state`) or isolate them in a `cove_*` extension object.
- Update repos to return internal domain records or minimal DB DTOs; routes and Gateway should serialize at the boundary.
- Add snapshot/contract tests for representative REST responses and Gateway events.

**Risk**: Medium. Existing frontend/plugin code may currently rely on minimal shapes and Cove-only fields being top-level. Adding fields is usually safe, but moving/removing extension fields would be breaking; prefer additive first.

**Test Impact**: Many API/Gateway tests will need expectation updates if they assert exact shapes. Additive fields should not break loose assertions, but snapshot tests would change.

**Effort**: M

**Priority**: P1

---

### Standardize Discord Error and Validation Envelopes

**What**: Replace ad hoc error responses with a central Discord-compatible error helper, including nested validation errors and correct status/code choices.

**Where**: `packages/server/src/validation.ts`, `packages/server/src/auth.ts`, `packages/server/src/routes/*.ts`, `packages/server/src/app.ts`.

**Why**: Cove returns `{ message, code }` in some places, bare `{ message }` in others, and some codes/statuses are off. Examples:
- `validationError()` returns only `{ message, code: 50035 }`, but Discord `50035 Invalid Form Body` responses commonly include an `errors` object describing field-level failures.
- Duplicate user creation returns `409` with code `10013 Unknown User`, which is semantically wrong.
- Non-member guild access is returned as `404 Unknown Guild`; depending on the compatibility target, Discord clients may expect `403 Missing Access (50001)` or `403 Missing Permissions (50013)` for known resources they cannot access.
- `DELETE /guilds/:guildId/members/:userId` returns `{ message: "Member not found" }` without a Discord code.
- OAuth/auth routes use different error formats from `/api/v10` routes.

**How** (high-level):
- Add a small error module, e.g. `discord/errors.ts`, with named helpers: `unauthorized`, `unknownGuild`, `unknownChannel`, `unknownMessage`, `unknownUser`, `missingAccess`, `missingPermissions`, `invalidFormBody`, `rateLimited`.
- Make every `/api/v10/*` route return a consistent Discord-style envelope.
- Extend validation helpers to collect field paths and emit a Discord-like `errors` object for `50035`.
- Decide and document whether Cove intentionally hides resource existence from non-members using `404`; if yes, make that a deliberate compatibility/security policy and apply it consistently.
- Keep non-Discord OAuth endpoints (`/api/auth/*`) separate, but avoid mixing them into Discord API helpers.

**Risk**: Medium. Existing tests assert current statuses and messages; changing 404 vs 403 semantics can break clients that rely on privacy-by-404. Make the policy explicit before changing access errors.

**Test Impact**: Error assertions across API tests will change. Add table-driven tests for every common error code/status pair.

**Effort**: M

**Priority**: P1

---

### Add a Real Permission/Ownership Layer Before More Mutating Endpoints

**What**: Introduce authorization checks for manage-channel, manage-guild/member, bot/user ownership, and self-only user updates.

**Where**: `packages/server/src/auth.ts`, `packages/server/src/routes/channels.ts`, `packages/server/src/routes/messages.ts`, `packages/server/src/routes/agents.ts`, likely DB schema/repositories for roles/permissions.

**Why**: Global auth currently means broad mutation power. Any authenticated member can:
- create/delete/patch channels,
- create/delete/regenerate tokens for users,
- patch/delete arbitrary users,
- add/remove guild members,
- bulk-delete all messages in a channel via `DELETE /channels/:id/messages`.

That is not Discord-compatible and is a security/design gap. Discord separates authentication from permissions: endpoint access depends on guild roles, channel permissions, bot ownership/scopes, and whether the target is `@me`.

**How** (high-level):
- Start with a minimal internal permission model rather than full Discord roles if needed:
  - `MANAGE_CHANNELS` for channel create/patch/delete.
  - `MANAGE_GUILD` or `MANAGE_MEMBERS` for member add/remove.
  - self-only rules for `PATCH /users/@me`; admin-only for agent/bot management extensions.
  - author-only or `MANAGE_MESSAGES` for message edit/delete; bulk delete should require `MANAGE_MESSAGES`.
- Add route guards such as `requireGuildPermission(repos, guildId, userId, permission)` and `requireChannelPermission(...)`.
- Treat current Cove-specific bot/agent administration endpoints as extension/admin endpoints, not general Discord user endpoints.
- Return `403` with `50013 Missing Permissions` or `50001 Missing Access` consistently.

**Risk**: High if turned on abruptly; current frontend/admin workflows may depend on every authenticated user being powerful. Roll out with a migration that grants existing admin/bot users appropriate permissions, or initially run in audit/log mode.

**Test Impact**: Many mutating-route tests must add admin/non-admin fixtures. Existing tests that use a generic token to mutate all resources will fail until updated.

**Effort**: L

**Priority**: P0

---

### Fill Core REST Endpoint Gaps and Correct Non-Discord Response Shapes

**What**: Add the highest-value missing Discord v10 endpoints and adjust existing endpoints whose response semantics differ from Discord.

**Where**: `packages/server/src/routes/channels.ts`, `packages/server/src/routes/messages.ts`, `packages/server/src/routes/agents.ts`, `packages/server/src/app.ts`, potentially new `routes/guilds.ts` and `routes/users.ts` split.

**Why**: A Discord-compatible client will expect more than the current subset. Important gaps or mismatches include:
- `GET /guilds/:guildId` is missing.
- `GET /guilds/:guildId/members/:userId` is missing.
- `GET /users/@me/guilds` exists twice (`agentRoutes` and `app.ts`) and should have one canonical implementation.
- `PATCH /users/@me` / `Modify Current User` is not separated from the Cove admin `PATCH /users/:id` path.
- `DELETE /channels/:id` returns `{ deleted: true }`, while Discord `Delete/Close Channel` returns the deleted channel object.
- `DELETE /channels/:id/messages` is a Cove-only delete-all operation; Discord bulk delete is `POST /channels/:id/messages/bulk-delete` with a message ID list and constraints.
- `POST /channels/:id/messages` accepts only `content`; many clients send/expect fields like `tts`, `embeds`, `allowed_mentions`, `message_reference`, `attachments`, `flags`, etc. Unsupported fields should either be safely ignored/documented or rejected with structured validation.

**How** (high-level):
- Create a compatibility matrix and implement the smallest useful Discord subset explicitly.
- Add `routes/guilds.ts` for guild and member read endpoints to avoid overloading `agents.ts`.
- Change `DELETE /channels/:id` to load and return the channel object after successful deletion.
- Deprecate Cove-only `DELETE /channels/:id/messages` or move it to a namespaced admin extension; add Discord-style bulk delete separately.
- Normalize duplicate route definitions so each Discord path has exactly one owner.

**Risk**: Medium. Returning a channel object on delete and moving delete-all behavior may break current Cove frontend code. Keep backward compatibility temporarily if needed, but do not present Cove-only routes as Discord-compatible.

**Test Impact**: Existing delete-channel/delete-all tests need updates or deprecation coverage. Add tests for new guild/member endpoints and duplicate-route removal.

**Effort**: M

**Priority**: P1

---

### Emit Complete Discord Gateway Events for REST Mutations

**What**: Dispatch the expected Gateway events and payloads for channel, guild member, and message lifecycle changes.

**Where**: `packages/server/src/routes/channels.ts`, `packages/server/src/routes/messages.ts`, `packages/server/src/routes/agents.ts`, `packages/server/src/ws/dispatcher.ts`, `packages/server/src/ws/session.ts`.

**Why**: Current event coverage is partial:
- Channel creation does not dispatch `CHANNEL_CREATE`.
- Channel deletion does not dispatch `CHANNEL_DELETE` and currently REST returns `{ deleted: true }`.
- Member add/remove emits Cove `GUILD_CREATE`/`GUILD_DELETE` only to the affected user, but does not broadcast `GUILD_MEMBER_ADD` / `GUILD_MEMBER_REMOVE` to guild members.
- `MESSAGE_DELETE` payload lacks `guild_id`; Discord events commonly include `id`, `channel_id`, and `guild_id?` for guild channels.
- `TYPING_START` payload lacks `guild_id` and `member`; it includes a Cove-only `username` field instead.
- Initial `READY` payload includes Cove `read_state` top-level, which is useful but not a Discord field; this should be namespaced or documented as a Cove extension.

**How** (high-level):
- Add dispatcher methods for `channelCreate`, `channelDelete`, `guildMemberAdd`, `guildMemberRemove`.
- Update REST mutating routes to dispatch the right event after successful repo mutation.
- Use the central serializers from the serializer proposal so Gateway event payloads and REST responses are consistent.
- Include `guild_id` on guild-scoped message/typing/delete events.
- Move Cove-specific `MESSAGE_ACK` and `read_state` to an extension contract (`COVE_MESSAGE_ACK`, `cove_read_state`, or similar) if strict Discord compatibility mode is needed.

**Risk**: Medium. More events can cause duplicate UI updates if current clients infer state from REST responses only. Client stores should be idempotent before enabling all lifecycle events.

**Test Impact**: Gateway dispatcher tests should expand to assert new events and guild isolation. Client subscription tests may need updates to handle additional event types.

**Effort**: M

**Priority**: P1
