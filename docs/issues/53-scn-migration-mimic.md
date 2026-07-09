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
     `treasures.appraised_at timestamptz` exists AND index `idx_treasures_appraised` exists.
   - `app-inventories` / `The inventory app breathes` — app probe script
     (`/usr/local/bin/inventory-check`, runs a SELECT using the new column) exits 0 (retries).
2. Dockerfile + boot:
   - Postgres (as in 51's install pattern, healthy this time — no startup fault); db `hoard`,
     user `mimicapp`; tables: `treasures(id, name)`, `schema_migrations(version int, dirty bool)`,
     `migration_locks(locked_by text, locked_at timestamptz)`.
   - Baked breakage (build-time SQL): `schema_migrations` rows: (1,false),(2,false),(3,**true**);
     `migration_locks` one stale row (`deployer-7`, old timestamp); column/index from v3 ABSENT.
   - Migration files present read-only at `/opt/app/migrations/00{1,2,3}_*.sql` — 003 adds the
     column + index (the intended change, discoverable); a `migrate` wrapper script exists but
     refuses while lock row exists / dirty=true (mimics real migrate tools' messages).
   - `inventory-check` + boot loop (app probe logging failures), boot-ok, sleep infinity.
3. Fix path: app error → psql exploration → discover dirty v3 + stale lock → read 003 SQL →
   apply its statements manually (or clear dirty+lock and re-run `migrate`) → both routes converge
   to identical final state; locks accept either.
4. Hints: 01 the app names a missing column; ask the database what IT thinks
   (`\d treasures`, select from schema_migrations) — ledger vs shape. 02 the migrate tool refuses:
   who holds the lock, and what does 003 actually do? read `/opt/app/migrations/003_*.sql`.
   03 near-answer: delete lock row, apply 003's ALTER/CREATE INDEX (IF NOT EXISTS), set v3 dirty=false.
5. `solution.md` (+ Lesson: ledger-truth drift, dirty flags, stale locks, idempotent repair
   (`IF NOT EXISTS`), verify with information_schema; dead end: DROP/re-CREATE table loses data —
   locks catch via row-count sentinel? add sentinel row + `shape-complete` includes
   `select count(*)>=3 from treasures`). `solution.sh`: psql heredoc doing the repair, verify probes.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed.
- [ ] Both fix routes (manual SQL vs unlock+rerun migrate) pass all locks — both transcribed.
- [ ] Destructive route (drop/recreate) fails `shape-complete` via the row-count sentinel — transcript.
- [ ] `ValidateDir` clean; image ≤ 700 MB (shares 51's postgres bulk; KU-5 table updated); no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=migration-mimic`; transcripts in PR.

## Dependencies

12, 29 (pattern-sibling: 51).

## Non-goals

Real migrate frameworks (golang-migrate etc. — the wrapper mimics their UX), multi-tenant schemas.

## Design References

DESIGN §14 Floor 5; COOKBOOK §4; KU-5.
