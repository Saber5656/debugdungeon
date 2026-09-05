# Title

Progress store and floor unlock rules

## Summary

Implement `internal/progress`: the `progress.json` store (DESIGN §8.2), clear/give-up merge
semantics (§9.5–9.7), floor unlock evaluation (§3.5), and totals.

## Context

Progress is the player's only durable achievement record; merge rules ("never downgrade", best-time
min-merge) and gating feed `list`, `map`, `play`, and later stats/achievements.

## Scope

- `internal/progress/store.go`, `internal/progress/unlock.go` + tests
- Not: run store (20), rendering (27/28)

## Detailed Requirements

1. Schema per DESIGN §8.2 (`schema_version: 1`); per-scenario record fields exactly as designed
   (`status new|cleared|given_up`, `clears`, `best_time_sec`, `hints_used_best`, `resets_best`,
   `first_cleared_at`, `last_played_at`) plus reserved `achievements` map (empty until 63; decode-tolerant).
2. Store: `Load` (missing → empty store; corrupt → quarantine `.corrupt-<ts>` + warn sentinel;
   `schema_version` > 1 → typed "upgrade debugdungeon" error, file untouched — §8.4),
   `Save` atomic 0600, `Mutate(paths, func(*Progress) error) error` — load→fn→save under a
   short-scoped flock on `progress.lock` (same pattern as issue 20's `WithLock`, but an
   independent lock file owned by this package — keeps 26 free of a dependency on 20).
3. `ApplyClear(id string, elapsedSec, hints, resets int, now time.Time) ClearResult` where
   `ClearResult{FirstClear, BestUpdated bool; PrevStatus string}` (consumed by victory banner 22
   and achievements 63):
   - status → `cleared` (from any); `clears++`; `best_time_sec = min(existing, elapsed)` (unset → set);
   - `hints_used_best`/`resets_best` update **only** when this clear set the new best time (companion stats of the best run);
   - set `first_cleared_at` once; always bump `last_played_at`; totals: `clears++`, `playtime_sec += elapsedSec`.
4. `ApplyGiveUp(id, now)`: status → `given_up` only when current is `new`; bump `last_played_at`.
5. Unlock rules (pure function over Registry + Progress):
   - `FloorUnlocked(floor int) bool`: floor 1 → true; floor N (2–5) → `clearsOnFloor(N-1) >= 2`;
     floor 6 (capstone) → **≥ 12 distinct registry scenarios with status `cleared`** (reclears of
     one room never count twice — §3.5).
   - `clearsOnFloor` counts **distinct cleared scenarios** on that floor (from the registry's
     floor mapping; scenarios present in progress but absent from registry are ignored).
   - `UnlockRequirementText(floor)` → exact strings used by `play` (exit 7) and `list`/`map`:
     `Clear 2 rooms on Floor N-1 to descend.` / `Clear 12 rooms in total to face the Cascade Throne.`
   - `free_roam` bypass is applied by callers (config), not inside the pure function.
6. Forward-compat policy (single rule, documented in the file header comment): unknown scenario
   **ids** are preserved (they're map keys of known-shaped records); unknown JSON **fields** at any
   level are dropped on the next save with a debug-log note. Schema growth happens via
   schema_version bumps, not field smuggling.

## Acceptance Criteria

- [ ] Merge tests: first clear, faster reclear (best updated + companions), slower reclear
      (best kept, clears++), give-up after clear (still cleared), clear after give-up (upgrades).
- [ ] Unlock matrix: boundaries (exactly 1 vs 2 clears on prior floor; 11 vs 12 total), floor 1 always, ignored-orphan progress entries.
- [ ] Corruption quarantine + fresh-start behavior.
- [ ] Atomic write + 0600 asserted.
- [ ] Requirement texts golden-tested (list/map/play reuse them verbatim).

## Validation

`go test ./internal/progress/...` green.

## Dependencies

05, 10.

## Non-goals

Run history (62), achievements evaluation (63), schema v2 migration machinery (single version now; §8.4 refusal rule only).

## Design References

DESIGN §3.5, §8.2, §8.4, §9.5–9.7.
