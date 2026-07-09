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

1. History file `<state>/history.jsonl` (0600), one JSON object per line:
   `{v:1, run_id, scenario_id, source, outcome:"cleared"|"given_up"|"abandoned", elapsed_sec, hints, resets, ended_at}`.
   - `AppendRun(paths, rec)` — O_APPEND single write (atomic enough per-line; partial trailing
     line tolerated on read: skip + debug-log).
   - Rotation: when > 10,000 lines at append time, keep newest 5,000 (rewrite via temp+rename).
   - Writers: victory (22) `cleared`, give-up (25) `given_up`, play `--force` abandon (21) `abandoned`.
2. `stats` command (read-only over history + progress + registry):
   - Overview block: rooms cleared X/Y, give-ups, total attempts (history lines), total time in
     dungeon (sum elapsed of cleared), hints used (sum), resets (sum).
   - Per-floor table: floor, cleared a/b, avg clear time, avg hints (cleared runs only).
   - Records block: fastest clear (room, mm:ss), most-attempted room, no-hint clears count.
   - Empty history → friendly "The chronicle is empty — go break something (then fix it)." exit 0.
3. All aggregate math table-driven-tested against a fixture history; unknown scenario ids in
   history (uninstalled packs) count in totals but not per-floor rows (documented).
4. Corrupt line policy: skip + count, `--verbose` prints skipped count.

## Acceptance Criteria

- [ ] Hooks fire exactly once per terminal state (unit: victory/give-up/abandon each append one line).
- [ ] Rotation at threshold proven (10k+1 → 5k newest kept, order preserved).
- [ ] `stats` golden: fixture with mixed outcomes + orphan pack ids + corrupt line; empty state.
- [ ] History file 0600; no Docker imports (works offline).
- [ ] Elapsed math consistent with §9.2 clock (history stores the same elapsed as the victory banner — cross-checked in an itest).

## Validation

`go test ./internal/progress/... ./internal/cli/...`; itest clear-one-room → stats reflects it.

## Dependencies

20, 22, 26 (25 for the give-up hook).

## Non-goals

Sharing/exports (v2: `stats --json` maybe), charts, streaks-by-date (no reliable local clock semantics worth it).

## Design References

DESIGN §3.4, §8.1; ADR-007; ISSUE_PLAN wave 9.
