# Title

Scenario: migration-mimic (Floor 5 — half-applied schema migration)

## Summary

Floor-5 room: a deploy died mid-migration — the migrations ledger claims v3 but the column it
adds is missing, and a stale advisory lock row blocks re-running; the player must reconcile ledger
vs reality and complete the migration by hand.

## Context

Teaches: schema-vs-ledger drift diagnosis, idempotent migration repair, advisory/lock-row
patterns, careful ALTER TABLE. Uses PostgreSQL (same base pattern as 51). DESIGN §14 Floor 5 row 3;
difficulty 4.

## Scope

`scenarios/migration-mimic/` only.

## Detailed Requirements

1. `scenario.yaml`: id `migration-mimic`, floor 5, difficulty 4, topics `[postgres, migrations, sql]`,
   time_estimate_min 40; resources `{memory_mb: 1024}`. Locks:
   - `ledger-honest` / `The ledger speaks truth` — SQL probe: `schema_migrations` has exactly
     versions 1,2,3 each `dirty=false`/no lock rows (`migration_locks` empty).
   - `shape-complete` / `The mimic's true shape is whole` — `information_schema.columns` shows
     `treasures.appraised_at timestamptz` exists AND index `idx_treasures_appraised` exists AND
     `select count(*) >= 3 from treasures` (seed-data sentinel — a drop/recreate "fix" loses the
     seeded rows and stays CLOSED; seed exactly 5 rows at build).
   - `app-inventories` / `The inventory app breathes` — app probe script
     (`/usr/local/bin/inventory-check`, runs a SELECT using the new column) exits 0 (retries).
2. Dockerfile + boot (C3; PostgreSQL via 51's exact Debian pattern — PGVER/pg_ctlcluster contract
   copied verbatim into this room's scripts; healthy server this time): db `hoard`, user
   `mimicapp` password `mimic-pass` (hba: scram for mimicapp@127.0.0.1); tables:
   `treasures(id int, name text)` seeded with 5 rows, `schema_migrations(version int, dirty bool)`,
   `migration_locks(locked_by text, locked_at timestamptz)`.
   - Baked breakage (build-time SQL): `schema_migrations` rows: (1,false),(2,false),(3,**true**);
     `migration_locks` one stale row (`deployer-7`, old timestamp); column/index from v3 ABSENT.
   - Migration files present read-only at `/opt/app/migrations/00{1,2,3}_*.sql` — 003 adds the
     column + index (the intended change, discoverable; written with `IF NOT EXISTS` forms).
   - `/usr/local/bin/migrate` wrapper — EXACT semantics (the review caught the dirty-rerun
     ambiguity): (a) any row in `migration_locks` → print `migrate: locked by <who> since <ts> —
     clear the lock first`, exit 1; (b) else if max(version) row has `dirty=true` → RE-APPLY that
     version's SQL file, then set `dirty=false`, print `migrate: repaired vN`, exit 0 (003's
     IF-NOT-EXISTS makes the re-apply idempotent); (c) else apply any missing versions in order;
     nothing to do → `migrate: up to date`.
   - `/usr/local/bin/inventory-check`: one SQL smoke exercising the v3 column
     (`PGPASSWORD=mimic-pass psql -h 127.0.0.1 -U mimicapp -d hoard -tAc "select count(appraised_at) from treasures"`)
     — exit 0 iff the query runs; boot loop runs it every 10s logging failures to
     `/var/log/inventory.log` (breadcrumbs), C3 boot-ok marker, sleep infinity.
3. Fix path: app error → psql exploration → discover dirty v3 + stale lock → read 003 SQL →
   apply its statements manually (or clear dirty+lock and re-run `migrate`) → both routes converge
   to identical final state; locks accept either.
4. Hints: 01 the app names a missing column; ask the database what IT thinks
   (`\d treasures`, select from schema_migrations) — ledger vs shape. 02 the migrate tool refuses:
   who holds the lock, and what does 003 actually do? read `/opt/app/migrations/003_*.sql`.
   03 near-answer: delete lock row, apply 003's ALTER/CREATE INDEX (IF NOT EXISTS), set v3 dirty=false.
5. `solution.md` (+ Lesson: ledger-truth drift, dirty flags, stale locks, idempotent repair
   (`IF NOT EXISTS`), verify with information_schema; dead-ends section: DROP/re-CREATE loses the
   seeded treasures → row-count sentinel stays closed). `solution.sh` (manual-SQL route): psql
   heredoc — `delete from migration_locks; ` + 003's IF-NOT-EXISTS statements +
   `update schema_migrations set dirty=false where version=3;` — then inventory-check verify loop.
   The wrapper route (`delete lock row via psql; migrate`) is the documented alternate; both end
   in identical state (lock-verified in the both-routes transcript).

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed.
- [ ] Both fix routes (manual SQL; unlock-then-`migrate` using wrapper semantics (b)) pass all locks — scripted transcripts `route-manual.txt` / `route-wrapper.txt` in the PR.
- [ ] Destructive route (drop/recreate) fails `shape-complete` via the row-count sentinel — transcript.
- [ ] `ValidateDir` clean; image ≤ 700 MB (shares 51's postgres bulk; KU-5 table updated); no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=migration-mimic`; transcripts in PR.

## Dependencies

12, 29 (51's PG pattern is copied, not imported — self-contained per ISSUE_PLAN's mutual-independence rule; values inlined above).

## Non-goals

Real migrate frameworks (golang-migrate etc. — the wrapper mimics their UX), multi-tenant schemas.

## Design References

DESIGN §14 Floor 5; COOKBOOK §4; KU-5.
