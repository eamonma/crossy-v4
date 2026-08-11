# DB isolation audit

2026-08-11. Scope: every Postgres touchpoint in both services, what isolation level each
transaction actually runs at, and whether the anomalies that level permits are reachable on
that path. Method: read of `packages/db`, `apps/api/src`, `apps/session/src`, `deploy/`, and
DESIGN.md §9, path by path.

## Verdict

Everything runs at Postgres default READ COMMITTED, and for this system that is the right
default. Concurrency correctness here deliberately does not live in the isolation level: it
lives in the per-game actor mailbox (a total order over gameplay writes), single-writer-per-
table enforced by role grants (INV-7), append-only logs, arbiter-based upserts (`ON CONFLICT`),
and the guarded `game_state` upsert. Those mechanisms make almost every path correct at READ
COMMITTED by construction, and a blanket bump to REPEATABLE READ or SERIALIZABLE would buy
nothing on the hot path while adding retry machinery (serialization failures, 40001) the
codebase has nowhere to put.

Three paths do reach anomalies READ COMMITTED permits. One of them can break a documented
invariant (DESIGN.md §7: a game is never left unadministrable) and deserves a fix; the other
two are contained and are judgment calls. None of them is best fixed by raising the global
isolation level; each wants a targeted guard or lock.

## Baseline facts

- No code sets an isolation level anywhere: no `SET TRANSACTION`, no Drizzle
  `isolationLevel` option, no `default_transaction_isolation` on the service roles
  (`deploy/provision.sh`, `deploy/bind-service-roles.sh`). Every `BEGIN` is a plain
  READ COMMITTED transaction. The session writer's `inTransaction`
  (`apps/session/src/writer.ts:266`) issues a bare `begin`; every `db.transaction` in the
  API passes no config.
- Both services connect through the Supabase SESSION pooler (`deploy/README.md:236`), which
  preserves normal transaction and session semantics. Migrations correctly refuse the
  transaction pooler because the Drizzle migrator takes session advisory locks and runs DDL
  (`packages/db/src/migrator.ts:17`, `deploy/README.md:243`). No isolation hazard here.
- The serialization work the database is NOT asked to do: gameplay writes are totally
  ordered by the actor mailbox (DESIGN.md §3), `game_state`/`cell_events`/`check_events`/
  `check_vote_events` have exactly one writing service by grant (INV-7), and the logs are
  INSERT-only by grant, so whole anomaly classes (lost update on the board, phantom events)
  are structurally off the table before isolation level even matters.

## Path-by-path

### Session writer flush: correct at READ COMMITTED (`apps/session/src/writer.ts`)

The flush transaction appends to the three logs with `ON CONFLICT DO NOTHING` (idempotent
retry after a partially observed flush) and upserts `game_state` behind the monotonic-seq /
terminal-status guard. The guard is sound at READ COMMITTED: `ON CONFLICT DO UPDATE ... WHERE`
evaluates against the current committed row after taking the row lock, so a stale writer is
refused (rowCount 0, `SnapshotRegressionError`) no matter how its snapshot interleaved. The
terminal flush reads `participantCount` and the event timestamps inside the same transaction
after its own appends; under the single-writer topology there is no concurrent writer to
those rows, so statement snapshots suffice. SERIALIZABLE here would add aborts and retries to
the hottest write path for zero additional correctness: the actor already provides the total
order, and the guard already catches the one scenario (deploy-overlap second writer) the
topology cannot exclude. DESIGN.md §9 calls the guard a tripwire, not coordination; READ
COMMITTED is enough for the tripwire to fire reliably. No change.

### Account deletion and host succession: the real gap (`apps/api/src/identity/deletion.ts`, `succession.ts`)

Two related anomalies, both requiring two account-scale mutations racing in one game.

**Lost promotion.** `succeedHost` SELECTs the earliest remaining solver, then UPDATEs that
row to host, with no `FOR UPDATE` and no rowCount check (`succession.ts:29`). If the chosen
successor concurrently deletes their own account (or is kicked), their membership DELETE can
commit between the select and the update. At READ COMMITTED the update blocks on the row
lock, re-evaluates after the delete commits, matches zero rows, and succeeds vacuously. The
deletion transaction then commits believing succession happened (`successions += 1`), no
auto-abandon is signaled, and the game is left with no host: exactly what DESIGN.md §7
promises never happens. Unlike the `game_state` guard there is no tripwire; the failure is
silent.

**Stale host set.** The "games this user hosts" read runs on the pool, outside the deletion
transaction (`deletion.ts:42`). A promotion landing between that read and the transaction
(the same double-deletion race, from the other side) leaves a hosted game out of
`hostGameIds`; the transaction deletes the membership rows anyway and the game is stranded
hostless with no successor and no abandon.

Likelihood at friends scale is very low; severity is an invariant break with no detection.
Recommended fix, in the codebase's existing idiom (explicit guards over optimistic retry):

1. Move the memberships read inside the transaction.
2. Take the game rows as the per-game mutex: `SELECT game_id FROM games WHERE game_id =
   ANY($hosted) ORDER BY game_id FOR UPDATE` at the top of the deletion transaction, and the
   same single-game lock at the top of the kick transaction (`games/routes.ts:1025`). Ordered
   locking keeps two racing deletions deadlock-free.
3. Check the UPDATE's rowCount in `succeedHost`; on 0, loop to the next candidate (the
   re-select at READ COMMITTED sees committed deletes) or fall through to auto-abandon.

SERIALIZABLE with a retry loop on these two transactions is a defensible alternative (SSI
detects both interleavings), but it introduces 40001 retry machinery for two cold paths that
a row lock solves in three lines. Step 3 alone closes the lost-promotion half and is nearly
free; do at least that.

### Join vs kick: contained write skew (`apps/api/src/games/routes.ts` `seatJoiner`, kick handler)

`seatJoiner` reads the denylist, then inserts the membership as separate statements on the
pool, not even one transaction. A kick (one transaction: membership DELETE plus denylist
INSERT) can commit between the two, leaving the kicked user both denylisted and re-seated:
the join's insert re-creates the row the kick just deleted. No isolation level on the kick
side alone can prevent this, since the join side is not a transaction.

Contained because the session re-checks the denylist on every connect
(`apps/session/src/repo.ts` `isDenied`), so live access stays refused; the residue is a stale
membership row (appears in rosters, can read the game view) until the host kicks again, which
self-heals (the second kick's denylist insert no-ops on conflict). The user held the invite
code already, so nothing new is exposed (and never solution content, INV-6 untouched).

Accept and document, or harden cheaply: after the seat insert, re-read the denylist and
delete the just-inserted row when denied (a compensating statement, no locks). Full closure
needs both paths to serialize on the game row (the same `FOR UPDATE` the succession fix
introduces) and is only worth it if roster-level integrity after a kick becomes a product
requirement.

### Role upgrade: one-word hardening (`apps/api/src/games/routes.ts:955`)

The spectator-to-solver upgrade reads the role, then runs `UPDATE ... SET role='solver'`
guarded only in application code. The write is unconditional on the row, so an interleaving
that promotes the same user to host between the read and the write (their own duplicate
upgrade request committing solver, then a racing deletion's succession promoting them) would
silently demote a host. It is a three-way race with negligible probability, but the fix is to
make the transition explicit where it executes: add `AND role = 'spectator'` to the UPDATE's
WHERE. Same idiom as the flush guard: the write itself states its precondition.

### Correct as-is at READ COMMITTED (no change)

- **Game creation** (`games/create.ts`): invite-code uniqueness by constraint plus retry on
  23505. The canonical pattern; no read-then-check gap.
- **Puzzle dedup** (`puzzles/routes.ts`): `ON CONFLICT (created_by, content_digest) DO
  NOTHING` then re-select. Arbiter-based, atomic under the two-tabs race, exactly as D23
  documents. `ON CONFLICT` waits out an uncommitted rival, so the re-select always finds the
  row (no delete path exists for puzzles; `games.puzzle_id` is RESTRICT).
- **Share-token mint** (`games/routes.ts:878`): targetless `ON CONFLICT DO NOTHING` (the
  partial unique "one active token per game" index is an arbiter) then re-select, with a
  defensive INTERNAL when a concurrent revoke empties the re-select. Right shape, including
  the defensive branch.
- **JIT identity upsert** (`auth/jit-upsert.ts`): single statement; the monotonic
  `is_anonymous` AND, the name CASE, and the avatar coalesce all evaluate on the locked
  current row, so READ COMMITTED gives lost-update freedom for free. The `xmax = 0` created
  signal is sound.
- **PATCH /me** (`identity/me.ts`): single-row last-write-wins by PK. Appropriate for user
  preference data.
- **Kick** (`games/routes.ts:1025`): membership DELETE plus denylist INSERT in one
  transaction; atomicity is what matters here (no "removed but not denied" half-state), not
  isolation.
- **Live Activity tokens** (`games/routes.ts:1124`): PK upsert, last write wins by design.
- **Reads** (GET /games, game view, analysis, session hydration and handshake):
  multi-statement reads without transactions, so each statement has its own snapshot.
  Cross-statement skew is possible and harmless: these gate or display, and skew is
  indistinguishable from the request arriving a moment earlier or later. The analysis
  endpoint additionally gates on completion, and a completed game is immutable (INV-4), so
  its multi-read is stable by construction. Wrapping these in REPEATABLE READ would cost a
  snapshot per request and change nothing observable.
- **Migrations** (`packages/db/src/migrator.ts`, deploy pipeline): advisory-locked,
  DDL-transactional, session-pooler-only by documented rule, destructive changes gated by the
  allowlist guard. Appropriate.

## Recommendations

1. **P1, fix**: succession locking and rowCount check as above (`deletion.ts`,
   `succession.ts`, kick handler). Defends the §7 never-unadministrable invariant. Test
   first (TDD, two-client interleaving in `api.test.ts` against real Postgres), name cites
   the invariant.
2. **P2, one line**: `AND role = 'spectator'` on the upgrade UPDATE.
3. **P3, judgment call**: join-vs-kick residue; document as accepted (the session connect
   gate is the enforcement point) or add the compensating re-check.
4. **Doc**: add one sentence to DESIGN.md §9 stating the isolation stance explicitly:
   default READ COMMITTED everywhere, correctness carried by structural serialization (actor
   total order, single-writer grants, arbiter upserts, guarded upsert), so nobody later
   assumes ambient SERIALIZABLE or "fixes" a race with a global bump.

Nothing here recommends changing the default isolation level. The architecture's bet, moving
serialization into the actor and the schema, is sound, and the audit confirms the database is
asked to do exactly the concurrency work READ COMMITTED plus constraints can do, everywhere
except the succession path.
