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

1. `Evaluate(ctx, deps, run) ([]locks.Report, bool, error)` — lock protocol (resolves the
   WithLock-vs-long-checks tension explicitly):
   - Under a SHORT `WithLock` section: Reconcile (20; broken → save state, release, return typed
     `ErrBroken` — CLI: guidance + exit 1), then transition running→checking + save, release.
   - Run `locks.RunAll` (19) WITHOUT holding the flock (may take minutes). Concurrent mutators are
     excluded by STATE, not the lock: `CanRun` returns `ErrBusy` for `checking` (20's eligibility
     table), so a second `check`/`hint`/`reset` gets the busy message.
   - Under a second SHORT section: save `last_lock_report`; not-all-open → back to `running`;
     all-open → leave state `checking` for `Victory` (same command invocation) to consume.
   - Returns reports + allOpen.
   - **Crash-during-RunAll recovery:** if the CLI dies while state is `checking`, the run is NOT
     wedged — the next command's `Reconcile` (20's crash-recovery rule) maps a `checking` run whose
     container is still `running` back to `running`. So a stuck `checking` state cannot outlive one
     command; no extra lease/deadline needed. (This is why `checking` is a transient state, never
     durable — DESIGN §7.3.)
2. `Victory(ctx, deps, run, reports) (Cleared, error)`:
   - Compute elapsed (20 helper), gather hints_revealed/resets.
   - `progress.ApplyClear` (26) → `ClearResult{FirstClear, BestUpdated, PrevStatus}`.
   - Ordered failure rules: progress-save FAILURE aborts (container and run kept — the player's
     clear must never be lost silently; retry via `check`). After progress succeeds: remove
     container (force; failure → warn, leftovers to `clean`), `Clear` run (failure → warn +
     instruct `give-up`), each under short WithLock.
   - Return `Cleared{ScenarioID string; Elapsed time.Duration; Hints, Resets int; FirstClear,
     BestUpdated bool; NextSuggestion string}` — NextSuggestion = next not-cleared unlocked
     scenario in registry order (may be empty).
3. `check` command:
   - No active run → exit 5. Wraps its mutations in `WithLock` (20) short sections — usable from a
     second terminal while `play` has the shell attached (DESIGN §3.3); a concurrent in-flight
     mutation surfaces 20's ErrBusy retry message.
   - Prints per-lock line: `Testing lock [name] … OPEN|CLOSED (msg)` (sanitized msg from 19).
   - All open → shared victory rendering (banner: `🏆 ESCAPED in mm:ss with N hints[, M resets]`,
     achievement hook point for 63 marked TODO) → exit 0.
   - Not all open → summary `k/n locks open` → exit 6.
4. Victory rendering lives in `internal/ui` as `RenderVictory(Cleared)`; this issue ALSO wires
   21's escape branch to `Evaluate`/`Victory` (replacing 21's stubs — the small `play.go` diff is
   in scope here), so both paths share one renderer (golden test). Repeat-clear banner:
   `Cleared again — new best!` when BestUpdated, else `Cleared again — best time kept.`
5. Idempotence: a second `check` after victory → exit 5 (run gone).
6. Failure atomicity: if progress save succeeds but container removal fails → warn (leftover
   reclaimed by `clean`), still Clear run (progress > tidiness).

## Acceptance Criteria

- [ ] Unit: Evaluate lock/state protocol (flock free during RunAll — concurrent WithLock succeeds
      mid-evaluation; second Evaluate blocked by `checking` state → ErrBusy); broken path saves
      state before returning; Victory ordering (progress before teardown; progress-save failure
      keeps container AND run; removal failure still Clears with warning).
- [ ] Golden: victory banners (first clear / repeat+best / repeat+kept).
- [ ] CLI: no-run → 5; partial locks → 6 with per-lock lines; all-open → 0.
- [ ] NextSuggestion logic covered (mid-floor, end-of-floor → next floor unlocked/locked, all-cleared → empty).
- [ ] itest (finalizes 21's loop test): with a fixture room, in-shell `escape` fail → fix (exec via docker) → `escape` → victory; progress.json shows cleared with plausible elapsed; two-terminal variant via `check` command.

## Validation

`make test` + `make itest` green; transcript of two-terminal flow (play in one, check in another) in PR.

## Dependencies

19, 20, 21 (stubs being replaced), 26.

## Non-goals

Achievements evaluation (63), stats history append (62 — adds a hook here), in-session loop (21).

## Design References

DESIGN §3.2, §5.1, §5.3 (codes 5/6), §9.4–9.7.
