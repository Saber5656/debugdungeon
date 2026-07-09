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

1. `Reset(ctx, deps, run) error` (valid from states `running`, `broken`):
   - transition → `resetting`, save.
   - Stop+remove old container (force; tolerate already-gone).
   - Image present? (`ImageList` by `run.ImageRef`) — normally yes; if pruned meanwhile →
     `EnsureImage` again (14) using the registry entry (content hash pinned by `run.ContentHash`;
     mismatch with current registry hash → refuse with "scenario content changed since this run
     started; give-up and start fresh" — protects §9 fairness).
   - `CreateRoom` + `InjectHelpers` + `StartRoom` (15/18) → update `container_id`,
     `resets++`, transition → `running`, save.
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

15, 20 (14 for the pruned-image path).

## Non-goals

Partial resets, snapshotting, keeping shell history across resets (HISTFILE dies with the container — documented).

## Design References

DESIGN §5.1, §7.3, §9.2–9.3, §11 F5.
