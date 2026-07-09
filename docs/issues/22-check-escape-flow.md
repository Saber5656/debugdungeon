# Title

`check` command and the escape/victory flow

## Summary

Implement lock evaluation as a shared game function, the standalone `debugdungeon check` command
(second-terminal path), and the victory flow that records progress and tears down the run.

## Context

`escape` (in-session) and `check` (outside) must be the same code path (DESIGN §3.3). Victory is
where run → progress handoff happens (§9.4–9.5).

## Scope

- `internal/game/evaluate.go`, `internal/game/victory.go`, `internal/cli/check.go` + tests
- Not: the session loop (21), progress store internals (26)

## Detailed Requirements

1. `Evaluate(ctx, deps, run) ([]locks.Report, bool, error)`:
   - Reconcile (20) first; broken → typed error (CLI: guidance + exit 1).
   - Transition running→checking, save; run `locks.RunAll` (19); save `last_lock_report`;
     transition back checking→running when not all open.
   - Returns reports + allOpen.
2. `Victory(ctx, deps, run, reports) (Cleared, error)`:
   - Compute elapsed (20 helper), gather hints_revealed/resets.
   - `progress.ApplyClear` (26) — min-merge best time, never-downgrade (§9.7).
   - Remove container (force), `Clear` run store, release lock.
   - Return `Cleared{ScenarioID, Elapsed, Hints, Resets, FirstClear bool, NextSuggestion string}`
     where NextSuggestion = next not-cleared unlocked scenario in registry order (may be empty).
3. `check` command:
   - No active run → exit 5. Acquires the run lock (20) — mutating (saves report/state).
   - Prints per-lock line: `Testing lock [name] … OPEN|CLOSED (msg)` (sanitized msg from 19).
   - All open → shared victory rendering (banner: `🏆 ESCAPED in mm:ss with N hints[, M resets]`,
     achievement hook point for 63 marked TODO) → exit 0.
   - Not all open → summary `k/n locks open` → exit 6.
4. Victory rendering lives in `internal/ui` as `RenderVictory(Cleared)` reused by 21's in-session
   escape path; identical output both paths (golden test).
5. Idempotence: a second `check` after victory → exit 5 (run gone).
6. Failure atomicity: if progress save succeeds but container removal fails → warn (leftover
   reclaimed by `clean`), still Clear run (progress > tidiness).

## Acceptance Criteria

- [ ] Unit: Evaluate state transitions incl. broken path; Victory ordering (progress applied
      before container removal; run cleared even when removal errors — assert warning surfaced).
- [ ] Golden: victory banner (first clear vs repeat clear differ: `Room cleared!` vs `Cleared again — best time kept`).
- [ ] CLI: no-run → 5; partial locks → 6 with per-lock lines; all-open → 0.
- [ ] NextSuggestion logic covered (mid-floor, end-of-floor → next floor unlocked/locked, all-cleared → empty).
- [ ] itest: with `_template` room, fail → fix (exec touch via docker) → check → victory; progress.json shows cleared with plausible elapsed.

## Validation

`make test` + `make itest` green; transcript of two-terminal flow (play in one, check in another) in PR.

## Dependencies

19, 20, 26.

## Non-goals

Achievements evaluation (63), stats history append (62 — adds a hook here), in-session loop (21).

## Design References

DESIGN §3.2, §5.1, §5.3 (codes 5/6), §9.4–9.7.
