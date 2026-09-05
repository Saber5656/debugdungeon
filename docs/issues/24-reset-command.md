# Title

`reset` command: rebuild the room, keep the run

## Summary

Implement `debugdungeon reset`: destroy the active room container and create a fresh one from the
same image, preserving run identity, revealed hints, and the clock (DESIGN §9.2–9.3).

## Context

Reset is the recovery path for self-inflicted damage and for `broken` rooms (F5/F7). Semantics
must match game rules exactly: resets are counted, the clock never resets.

## Scope

- `internal/game/reset.go`, `internal/cli/reset.go` + tests
- Not: `play` resume logic (21), image rebuild policy (never rebuilds — ADR-004)

## Detailed Requirements

1. `Reset(ctx, deps, run) error` (valid from states `running`, `broken` via `CanRun`; state
   mutations under short `WithLock` per 20):
   - transition → `resetting`, save.
   - Stop+remove old container (force; tolerate already-gone).
   - Image present? (`ImageList` by `run.ImageRef`) — normally yes; if pruned meanwhile → resolve
     the scenario via the registry (`ByID(run.ScenarioID)`; missing → refuse: "scenario no longer
     installed — give-up to end this run") and compare hashes: registry hash ≠ `run.ContentHash` →
     refuse with "scenario content changed since this run started; give-up and start fresh"
     (protects DESIGN §9 rule-2/3 fairness); equal → `EnsureImage` (14).
   - `CreateRoom` (15) → save container_id → `InjectHelpers` (18) → `StartRoom` → `resets++`,
     transition → `running`, save (20's sequencing rule).
   - **Failure mid-reset** (any step after the old container is gone): save state `broken` with
     the error logged; the run is NOT lost — the player retries `reset` or `give-up` (F5 path).
     A crash mid-reset leaves state `resetting`, which `Reconcile`-at-startup maps to `broken`.
2. CLI: requires active run (exit 5), acquires run lock, y/N confirm
   (`This wipes the room's state (hints and the clock are kept). Continue?`; `--yes` skips),
   prints `The room reforms around you… (reset #N)` then resume guidance (`debugdungeon play <id>`).
3. Reset does NOT attach a session (player runs `play` to re-enter) — keeps command composable
   from a second terminal while a dead session's terminal recovers.
4. Note in code + docs: a session attached in another terminal dies when its container is removed;
   session (17) surfaces this as Broken — acceptable, documented (DESIGN §3.3).

## Acceptance Criteria

- [ ] Unit: state transitions (running→resetting→running; broken→resetting→running); counter increments; hints/started_at untouched (assert unchanged fields).
- [ ] Unit: pruned-image path calls EnsureImage; changed-hash path refuses with the specified message.
- [ ] CLI: no run → 5; confirm-decline aborts with no changes.
- [ ] itest: create room, write a file inside, reset, file gone, same run_id, resets==1.

## Validation

`make test` + `make itest` green.

## Dependencies

10, 14, 15, 18, 20.

## Non-goals

Partial resets, snapshotting, keeping shell history across resets (HISTFILE dies with the container — documented).

## Design References

DESIGN §5.1, §7.3, §8.3–8.4, §9 (rules 2–3), §11 F5/F7.
