---
status: descriptive
verified: 8eced56
---

# @crossy/session

The session service (DESIGN.md section 3, section 6): the stateful WebSocket tier. One
in-memory actor per live game is the single writer for its `game_state`, `cell_events`,
`check_events`, and `check_vote_events`. Handshake, mailbox, write-behind flush,
reconnect resync, and SIGTERM drain are the core; this file also covers the membership
lifecycle, Live Activity push, the session half of the check vote, analytics, and the
service's logging posture.

## Endpoints

- `wss://{host}/games/{gameId}/ws`: the gameplay socket (PROTOCOL.md section 2). Handshake
  check order is fixed: version, token, game, denylist, then membership, with the denylist
  strictly before membership so a kicked user gets `DENIED`, not `NOT_PARTICIPANT`.
- `GET /` (and any other path): a plain `ok` health probe.
- `POST /internal/games/{gameId}/membership-changed`: the internal membership signal
  (DESIGN.md section 6). See below.
- `POST /internal/games/{gameId}/live-activity-registered`: the Live Activity welcome
  notice (PROTOCOL.md section 12a). The API POSTs here after a token registers, so the
  emitter hands the fresh island the current authoritative frame at once. Same static
  bearer and private-port routing as membership-changed.

## The membership-changed internal endpoint (DESIGN.md section 6, INV-8)

The API is the single writer on memberships and the denylist. After it commits a change it
calls this endpoint so a live actor enforces the new authoritative state. The session
verifies, it never mutates (INV-8):

- The request body is only a hint. Shape:
  `{ "change": "kick" | "role" | "abandon", "userId"?: string, "by"?: string }`. For a kick
  or role change the session re-reads membership and the denylist from Postgres and acts on
  that, so the body cannot assert a membership fact.
- Kick or role change touches only the live actor's connected sockets: a denied user is sent
  `kicked` and closed 1008, and the rest have their cached role refreshed. A passivated game
  has no live actor, so the call is a no-op (the denylist plus connect-time re-verify enforce
  it at the next connect); this path never hydrates.
- Abandon hydrates the actor on demand, since only the actor may write `game_state`, and
  emits and synchronously flushes `gameAbandoned` before broadcast. Abandon on an already
  terminal game is a no-op (INV-4).

### Authentication and the static bearer

The endpoint is bearer-authenticated with a static secret shared with the API. It arrives via
config, never hardcoded:

- **`INTERNAL_BEARER_TOKEN`** (env): the static internal bearer. The endpoint requires
  `Authorization: Bearer $INTERNAL_BEARER_TOKEN`. When the variable is unset the endpoint is
  disabled and returns 503, so a misconfigured deploy fails closed rather than serving the
  endpoint unauthenticated. The comparison is constant-time.

Responses: 200 `{ ok: true }` on success, 401 (no bearer), 403 (wrong bearer), 400 (malformed
body), 503 (endpoint not configured), 500 (internal fault).

The bearer is defense-in-depth on an already-private channel (DESIGN.md section 6, section
15). Its blast radius stays a forced re-verification, disconnect, or abandon, never data
access, because the actor re-reads authoritative state from Postgres and treats the body only
as a hint.

## Live Activity push (`src/push/`)

The session pushes iOS Live Activity updates over APNs as a game progresses (PROTOCOL.md
"Live Activity push"). The emitter (`emitter.ts`) turns actor events into frames, `roster.ts`
tracks the registered per-activity update tokens for a game, `policy.ts` decides when a frame
is worth a push (rate and change gating), `tokens.ts` reads the tokens the API registered, and
`apns.ts` is the APNs adapter over HTTP/2. It is built only when the APNs env is complete;
otherwise the emitter is inert and the rest of the service runs unchanged, so dev and CI never
touch APNs.

## Analytics (`src/analytics/`)

A noop-by-default PostHog port, enabled by `POSTHOG_TOKEN`; the event vocabulary is
ANALYTICS.md. Properties are flat ids and counts, so board content structurally cannot
ride an event (INV-6).

## The check vote (D32)

The engine owns the vote state machine (`applyWithVote`); the session owns what the engine
may not touch: clock, presence, persistence. PROTOCOL.md section 10 and
design/check-vote/UX.md are the law.

- **Timebox.** `CHECK_VOTE_TTL_MS = 30_000`, session-owned (`actor.ts`). When a vote opens
  the actor stamps `expiresAt` = now + TTL onto the broadcast and the snapshot and arms a
  timer; expiry reaches the engine as an `expireCheckVote` input, since the engine models
  no clock (INV-9). Per-actor injectable via `ActorOptions.checkVoteTtlMs`, so tests
  exercise expiry without a real 30 s wait.
- **Electorate.** Frozen at proposal accept time from the live host and solver connections
  on the actor (always including the proposer, ascending ASCII, spectators never vote) and
  handed to the engine as data on the command.
- **Errors.** `VOTE_PENDING`, `NO_VOTE_OPEN`, `NOT_ELECTOR`, `ALREADY_VOTED` (PROTOCOL.md
  section 11), each non-fatal and carrying the offending commandId.
- **Persistence.** Vote lifecycle rows buffer under the write-behind and flush to the
  append-only `check_vote_events` log exactly like `check_events` (`writer.ts`, migration
  0014). The open vote rides every snapshot: `checkVote` on the section 4 board payload and
  on the persisted `game_state` row.
- **Crash rehydrate.** A hydrated vote whose deadline already passed closes failed
  `EXPIRED` with no broadcast (the welcome snapshot heals it); the flush is posted through
  the mailbox. A vote still in the future re-arms the timer for the remaining time.

## Logs and diagnostics (Track D)

Structured lines carry ids, codes, and counts only, never cell values (INV-6):

- Every socket close emits one line: gameId, userId (or `pre-handshake`), the close code,
  `socketAgeMs`, `livenessFired`, and `wasLast` (whether the close emptied the actor). A
  distinct line marks a liveness-timer reap (`server.ts`).
- The inline submit path catches flush rejections: the buffer is retained and the actor and
  socket stay up, because one process hosts many games and a Postgres fault on one must
  never fault the process. Genuinely unknown faults still fail fast:
  `unhandledRejection` and `uncaughtException` handlers log the stack and exit 1
  (`main.ts`), so Railway restarts the service and the crash is never blind.

## Configuration

Read only in `main.ts` (12-factor), passed to `createSessionServer`:

| Variable                                            | Required | Purpose                                                                                                |
| --------------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| `DATABASE_URL`                                      | yes      | Postgres connection (the `crossy_session` role in production).                                         |
| `SUPABASE_ISSUER`                                   | yes      | Token issuer for the auth port (JWKS derived from it).                                                 |
| `INTERNAL_BEARER_TOKEN`                             | no       | Enables the internal endpoints (membership-changed, live-activity-registered); unset disables them.    |
| `PORT` / `HOST`                                     | no       | Listen address (defaults 8081 / 0.0.0.0).                                                              |
| `INTERNAL_PORT`                                     | no       | Separate private port for `/internal` (Railway deploy); unset serves `/internal` on the public `PORT`. |
| `APNS_TEAM_ID` / `APNS_KEY_ID` / `APNS_PRIVATE_KEY` | no       | APNs credentials for the Live Activity push emitter; with any absent the emitter is inert.             |
| `PASSIVATE_AFTER_MS`                                | no       | Override the idle-passivation window (DESIGN.md section 15); unset uses the server default.            |
| `POSTHOG_TOKEN`                                     | no       | Enables product analytics (posthog-node); unset selects a noop and the SDK is never constructed.       |
| `POSTHOG_HOST`                                      | no       | PostHog ingestion host; defaults to `https://us.i.posthog.com`.                                        |

## Testing

`vitest run`. The integration suite boots a Testcontainers Postgres, applies the committed
migrations, and drives the real server over real `ws` sockets and the real internal endpoint.
The server pool runs as the least-privilege `crossy_session` role, so the grants are exercised
for real: the session can write `game_state` and `cell_events` but is provably denied writes to
`memberships` and `game_denylist` (INV-7, INV-8). Auth is the in-memory fake; no suite touches a
network.
