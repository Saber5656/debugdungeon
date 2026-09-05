# Title

Run history recording and `stats` command

## Summary

Append a compact record of every finished run to a local history file and implement
`debugdungeon stats`: aggregate play statistics across the dungeon.

## Context

DESIGN §3.4 tracks per-scenario bests in progress.json; richer aggregates (totals, averages,
streak-ish facts) need an append-only history. Feeds achievements (63). Zero telemetry — all local
(ADR-007).

## Scope

- `internal/progress/history.go`, `internal/cli/stats.go` + tests
- Hooks in victory (22) and give-up (25) flows — one function call each
- Not: achievements (63), any upload/share

## Detailed Requirements

1. History file `<state>/history.jsonl` (0600; path = `Paths.HistoryFile` from 05), one JSON
   object per line — `RunRecord` contract (types explicit):
   `{V int(=1); RunID, ScenarioID, Source string; Outcome string("cleared"|"given_up"|"abandoned");`
   `ElapsedSec int (run-start→terminal for ALL outcomes, §9 rule 2 clock); Hints, Resets int;`
   `EndedAt string (RFC3339 UTC)}`. Unknown/invalid Outcome or negative numbers on read → line
   skipped as corrupt.
   - `AppendRun(paths, rec)` — O_APPEND single write. Concurrency: all three writers already
     execute inside the terminal-state flows which run under `game.WithLock` (20) — appends and
     the rotation rewrite happen under that lock; `stats` reads a snapshot without the lock
     (torn/partial trailing line tolerated: skip + debug-log).
   - Rotation: when > 10,000 lines at append time, keep newest 5,000 (temp+rename, under the lock).
   - Writers: victory (22) `cleared`, give-up (25) `given_up`, play `--force` abandon (21) `abandoned` — one call each.
2. `stats` command (read-only over history + progress + registry) — deterministic rules:
   - Overview block: rooms cleared X/Y (X = distinct cleared ids present in the registry; Y =
     registry size), give-ups (count of given_up records), total attempts (valid history lines),
     total time (sum ElapsedSec of cleared records), hints used (sum, all records), resets (sum).
   - Per-floor table: floor, cleared a/b (distinct/registry), avg clear time + avg hints over
     cleared records of that floor (missing data → `—`).
   - Records block: fastest clear (min ElapsedSec; tie → lexically first scenario id),
     most-attempted room (max record count; same tie rule), no-hint clears count.
   - Scenario titles rendered from the REGISTRY (sanitized per C7); history-only orphan ids
     (uninstalled packs) count in overview totals, are excluded from per-floor and records blocks
     (documented in the command help).
   - Empty history → friendly "The chronicle is empty — go break something (then fix it)." exit 0.
3. All aggregate math table-driven-tested against a fixture history (mixed outcomes, orphan ids,
   corrupt line, tie cases).
4. Corrupt line policy: skip + count, `--verbose` prints skipped count.

## Acceptance Criteria

- [ ] Hooks fire exactly once per terminal state (unit: victory/give-up/abandon each append one line).
- [ ] Rotation at threshold proven (10k+1 → 5k newest kept, order preserved).
- [ ] `stats` golden: fixture with mixed outcomes + orphan pack ids + corrupt line; empty state.
- [ ] History file 0600; no Docker imports (works offline).
- [ ] Elapsed math consistent with DESIGN §9 rule 2 (history stores the same elapsed as the victory banner — cross-checked in an itest).

## Validation

`go test ./internal/progress/... ./internal/cli/...`; itest clear-one-room → stats reflects it.

## Dependencies

20, 21 (abandon hook), 22 (victory hook), 25 (give-up hook), 26.

## Non-goals

Sharing/exports (v2: `stats --json` maybe), charts, streaks-by-date (no reliable local clock semantics worth it).

## Design References

DESIGN §3.4, §8.1; ADR-007; ISSUE_PLAN wave 9.
