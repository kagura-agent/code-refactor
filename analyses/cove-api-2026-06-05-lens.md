# Cove REST API + Gateway — Discord v10 Compatibility Audit
**Analyst:** 🔬 Lens · **Date:** 2026-06-05
**Scope:** `packages/server/src/{app.ts, routes/, ws/, auth.ts, validation.ts}` + `packages/shared/src/types.ts`

I read every file in scope and cross-referenced against Discord API v10 (REST `/api/v10/*` and Gateway `?v=10&encoding=json`). Cove is broadly Discord-shaped — paths, snowflakes, opcode numbering, READY layout, MESSAGE_* events all line up. What follows are the gaps a real Discord-compatible client (discord.js, eris, custom bot) will hit, plus design-level bugs that pre-date the compatibility question.

---

### 1. Message & channel mutation lacks author/permission checks
**What:** `PATCH/DELETE /channels/:id/messages/:msgId` and `PATCH/DELETE /channels/:id` authorize on "is guild member" only — any member can edit/delete any other member's message, or rename/delete any channel.
**Where:** `routes/messages.ts` (PATCH/DELETE message handlers), `routes/channels.ts` (PATCH/DELETE channel handlers).
**Why:** Cross-tenant write authority. Discord requires the message author for edit, and `MANAGE_MESSAGES` for cross-author delete; channel mutation requires `MANAGE_CHANNELS`. Cove enforces neither. This is the most consequential finding — it is exploitable today by any authenticated guild member.
**How:** Introduce a minimal permission layer:
- Message PATCH → require `message.author.id === botUser.id`, else 403 `{ message: "Missing Permissions", code: 50013 }`.
- Message DELETE → allow self OR a role flag (start with `is_owner` / `bot` until roles land).
- Channel PATCH/DELETE → gate on guild-owner or an admin flag.
Centralize as `requireAuthor(repo, msg, user)` / `requireGuildAdmin(repo, guild, user)` helpers reused by routes.
**Risk:** Existing UI/clients that rely on the permissive behavior will break — needs a coordinated client update.
**Test Impact:** New 403 cases must be added; some existing tests asserting 200 on cross-author edits will need to flip to 403.
**Effort:** M · **Priority:** P0

---

### 2. Validation error envelope is not Discord 50035-shaped
**What:** `validationError()` returns `{ message, code: 50035 }`. Discord's 50035 ("Invalid Form Body") always includes a structured `errors` map: `{ code: 50035, message: "...", errors: { content: { _errors: [{ code: "BASE_TYPE_REQUIRED", message: "..." }] } } }`.
**Where:** `validation.ts` (envelope), all routes that call `validationError()`. Also `routes/auth.ts` and `routes/register.ts` return `{ message }` with **no `code` at all** on 400s (e.g. "Missing authorization code", "Invalid pending token", "inviteCode and pendingToken are required") — Cove's own error contract is inconsistent across files.
**Why:** Discord clients (discord.js `DiscordAPIError`) parse `errors` to surface field-level validation; absence breaks UX and any `error.code` dispatch logic. Inconsistent shape across routes also breaks shared client error handlers.
**How:**
- Extend validators to collect `{field, code, message}` tuples instead of a single string.
- `validationError(c, errors[])` emits `{ code: 50035, message: "Invalid Form Body", errors: {<field>: { _errors: [...] }} }`.
- Audit non-validation 4xx responses to always include a Discord error code (`40001`, `10003`, `10004`, `10008`, `10013`, `50013`, `50035`, `50007`, etc.). The "Member not found" 404 in `agents.ts` and the register/auth 400s should pick concrete codes.
**Risk:** Low — additive shape change; clients ignoring `errors` continue to work.
**Test Impact:** Existing tests that assert `{ message, code: 50035 }` literal need to be widened.
**Effort:** S · **Priority:** P0

---

### 3. Gateway: opcode-4 collision, missing RESUME, missing `resume_gateway_url`
**What:** Three Gateway-level deviations:
1. `GatewayOpcode.REQUEST_TYPING = 4` collides with Discord's reserved op `VOICE_STATE_UPDATE = 4`. Any real Discord client that ever sends voice state update will be interpreted as "request typing" (and vice versa).
2. `RESUME = 6` is declared in the enum but never handled in `ws/index.ts` — there is no resume path, no replay buffer, no `INVALID_SESSION (op 9)`, no `RECONNECT (op 7)`. Clients that drop and resume will silently lose events.
3. `HELLO` does not include `resume_gateway_url`, and `READY.d` is missing it too. Since Gateway v10, clients use `resume_gateway_url` from READY to reconnect — without it, discord.js falls back to the original URL but logs a warning and resume cannot be tested.
Additional minor: IDENTIFY does not validate the `intents` field (required since v8); `HEARTBEAT` doesn't read the client-sent last-seq `d`.
**Where:** `packages/shared/src/types.ts` (enum), `ws/index.ts` (opcode dispatch + HELLO payload), `ws/session.ts` (READY payload, seq tracking).
**Why:** Op-4 collision is a wire-protocol footgun that's invisible until a real Discord client connects. Missing RESUME makes the gateway non-recoverable from any network blip — every disconnect forces a full READY replay, defeating the design intent of the Gateway protocol. Cove already increments `seq` per dispatch, so the missing piece is just a bounded ring-buffer keyed by `session_id`.
**How:**
- Move `REQUEST_TYPING` to a non-reserved opcode (Discord opcodes used: 0,1,2,3,4,6,7,8,9,10,11,13). Pick **op 14** or higher and document as Cove extension. Old clients break — coordinate with Cove client.
- Implement RESUME: on IDENTIFY store `{session_id → recent dispatches[last 250]}`, on RESUME (op 6) with valid `session_id`+`seq` replay missing dispatches; on mismatch send INVALID_SESSION (op 9, `d=false`).
- Add `resume_gateway_url` to READY (`ws://host/gateway` is fine for now).
- Use HEARTBEAT `d` as last-received seq for liveness diagnostics.
**Risk:** RESUME is the largest item — needs careful per-session memory cap. Op-4 move is a breaking change for the Cove web client.
**Test Impact:** New WS tests for resume/replay; existing tests that hardcode op 4 for typing need updating.
**Effort:** L (RESUME) + S (others) · **Priority:** P0 for op-4 collision and `resume_gateway_url`, P1 for full RESUME.

---

### 4. Bulk-delete and channel-delete response shapes diverge from Discord
**What:** Two non-Discord response shapes:
- `DELETE /channels/:id/messages` (delete all in channel) returns `{ deleted: <count> }`. Discord's bulk delete is `POST /channels/{channel.id}/messages/bulk-delete` with body `{ messages: [ids] }` returning `204`. There is no "delete all" endpoint in Discord at all.
- `DELETE /channels/:id` returns `{ deleted: true }`. Discord returns the deleted channel object (status 200).
**Where:** `routes/channels.ts`, `routes/messages.ts`.
**Why:** Discord-compatible clients call `channel.delete()` and expect to receive the channel object back to fire local cache invalidation; `{ deleted: true }` causes type/parsing errors. Bulk-delete clients call `bulkDelete([ids])` against a URL Cove doesn't expose.
**How:**
- `DELETE /channels/:id` → return the deleted channel object (200).
- Add `POST /channels/:id/messages/bulk-delete` accepting `{ messages: string[] }` (Discord caps at 100 ids ≤ 2 weeks old; for Cove drop the age limit, keep the cap). Keep or remove the existing `DELETE /channels/:id/messages` "delete all" as a Cove extension — but mark it explicitly non-Discord in the route comment.
**Risk:** Low; existing internal callers may need a one-line update.
**Test Impact:** Update channel-delete tests; add bulk-delete tests.
**Effort:** S · **Priority:** P1

---

### 5. Rate-limit envelope is entirely absent
**What:** No 429 path, no `X-RateLimit-Limit/Remaining/Reset/Reset-After/Bucket` headers, no `Retry-After`. Discord clients build their rate-limit queue from these headers; without them they may flood Cove and never back off.
**Where:** `app.ts` (middleware), all routes.
**Why:** discord.js / eris key their internal rate limiter on `X-RateLimit-Bucket` per route. Cove sending zero rate-limit headers is interpreted as "unlimited", so on a slow Cove instance a misbehaving bot can wedge the server with no protocol-level brake.
**How:** Add a Hono middleware backed by an in-memory token bucket keyed by `(route-template, user.id)`. Emit `X-RateLimit-*` on every authenticated response, return 429 with `{ message: "You are being rate limited.", retry_after: <sec>, global: false, code: 0 }` plus headers when exceeded. Pick conservative limits (50 req/sec global, 5/sec per channel-write) — they can be raised once telemetry exists.
**Risk:** Misconfigured limits cause user-visible throttling. Ship behind an env flag for staging dogfood first.
**Test Impact:** Add 429 path tests; existing tests unaffected if limits set high.
**Effort:** M · **Priority:** P1

---

### 6. OAuth token leaks via redirect query string
**What:** `routes/auth.ts` callback redirects to `/?token=<bearer>` and `/?pending=<token>`. URL query strings appear in browser history, server access logs, `Referer` headers to any third-party resource the SPA loads, and shared screenshots.
**Where:** `routes/auth.ts` lines `return c.redirect(\`/?token=${token}\`)` and `\`/?pending=${pendingToken}\``.
**Why:** This is a credential-exposure bug independent of Discord compat — the token is a long-lived bearer that authenticates every REST call. Any analytics pixel or external avatar request from the landing page will leak it to a third party via Referer.
**How:** Two options:
- Set the token as an `HttpOnly; Secure; SameSite=Lax` cookie and redirect to `/`. The SPA reads the user via `/api/auth/me` and uses the cookie automatically.
- Or, keep redirect but use the URL fragment (`/#token=…`) — fragments are not sent in Referer or logged server-side. Cookie is the right answer long-term.
**Risk:** Cookie path needs CSRF protection (SameSite=Lax mostly covers it; add CSRF token for state-changing routes if browsers ever weaken SameSite default).
**Test Impact:** Auth E2E flow needs adjustment; SPA token handling changes.
**Effort:** S (fragment) / M (cookie + CSRF) · **Priority:** P1

---

### 7. Cove invents user CRUD that Discord doesn't expose
**What:** `routes/agents.ts` exposes `POST /users`, `PATCH /users/:id`, `DELETE /users/:id`, `POST /users/:id/token`. Discord has none of these via REST — users are created via OAuth, edited via `PATCH /users/@me` only, never deleted.
**Where:** `routes/agents.ts`.
**Why:** A Discord-compatible client calling `client.users.fetch(id)` works (Cove implements `GET /users/:id`), but anything writing to `/users` is Cove-specific. More importantly: any authenticated user can `DELETE /users/<someone-else>` or rotate someone else's bot token via `POST /users/:id/token` — there is no actor==target check. This is a second authz hole on top of #1.
**How:**
- Add ownership/admin checks on `PATCH`, `DELETE`, `POST /:id/token` — actor must be the target user OR an admin role.
- Move the agent-management endpoints under a clearly-namespaced prefix (e.g. `/api/v10/cove/agents`) and leave only `GET /users/:id`, `PATCH /users/@me`, `GET /users/@me` under the Discord-compatible namespace. Document the `/cove/*` prefix as the Cove extension surface.
**Risk:** Existing Cove web client likely calls these paths — coordinate move.
**Test Impact:** Path-rewriting in tests; new 403 cases.
**Effort:** M · **Priority:** P0 (auth gap) / P1 (namespace cleanup)

---

## Quick wins not promoted to top-7
These are 1-line fixes worth bundling:
- `PUT /guilds/:guildId/members/:userId` returns existing member as `200`; Discord returns `204 No Content` for existing.
- `GET /api/v10/gateway` returns `{ url }`; for bot tokens Discord adds `{ shards, session_start_limit: { total, remaining, reset_after, max_concurrency } }`. Stub with sane defaults.
- `READY.d` is missing Discord-expected fields `application`, `user_settings`, `relationships`, `private_channels`. Most clients tolerate absence; discord.js handles missing fields. Low priority but worth a one-line `application: { id: <bot-id> }` stub.
- No `OPTIONS` / CORS middleware visible in scope — verify it lives in the server entry; if not, Discord-compatible browser clients (none today, but the Cove web app) will fail preflight on cross-origin.
- `agents.ts` `GET /users/:id` lacks `auth` middleware — currently saved by the global `/api/*` middleware, but the route is registered without it, so any future refactor that splits routing could expose it. Add `auth` for defense in depth.

## Summary
| # | Title | Priority |
|---|---|---|
| 1 | Author/permission checks on message & channel mutation | **P0** |
| 2 | Discord 50035 error envelope + consistent error codes | **P0** |
| 3 | Gateway op-4 collision, RESUME, `resume_gateway_url` | **P0** (op-4) / P1 (RESUME) |
| 4 | Channel-delete & bulk-delete response shapes | P1 |
| 5 | Rate-limit headers & 429 envelope | P1 |
| 6 | OAuth token leak via redirect query string | P1 |
| 7 | User-CRUD authz gap + namespace cleanup | P0 / P1 |

The three P0s (authz on writes, error envelope, gateway op-4) can ship independently and unblock real Discord-client interop. Everything else is independently mergeable.
