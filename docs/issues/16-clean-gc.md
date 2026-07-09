# Title

`clean` command and orphan garbage collection

## Summary

Implement discovery and removal of DebugDungeon-managed Docker leftovers: `debugdungeon clean`
(containers) and `clean --all` (also images), sparing whatever the active run references.

## Context

Every abnormal exit can strand a container; images accumulate per content-hash (ADR-004
consequence). DESIGN §7.8 and failure F10.

## Scope

- `internal/dockerx/gc.go`, `internal/cli/clean.go`
- Not: automatic background GC, run-store internals (20)

## Detailed Requirements

1. `ListManaged(ctx, api) (Managed, error)` — containers (all states) and images filtered by label
   `com.debugdungeon.managed=true`; images include size; result sorted stable.
2. `Clean(ctx, api, protected []string, all bool, confirm func(prompt string) bool) (Report, error)`:
   - Containers not in `protected`: stop (5s) if running, remove force. Protected ones listed as "kept (active run)".
   - `all`: additionally remove managed images (`ImageRemove` force, prune children) after a
     single confirmation showing count + total size. Non-TTY without `--yes` → abort with exit 1.
   - `Report{RemovedContainers, KeptContainers, RemovedImages, FreedBytes}`.
3. `clean` command wiring:
   - Protected set: read the active run's container ID via `game.ActiveContainerID(paths)` — a
     minimal helper INTRODUCED HERE in `internal/game/active.go` that reads `run.json` and returns
     `(containerID, scenarioID, ok)` using a 3-field anonymous struct decode. Issue 20 replaces its
     internals with the real store, keeping the signature (documented in code comment).
   - Flags: `--all`, `--yes` (skip confirmation).
   - Output: table of actions + `freed ~X.Y GB` for images.
4. `doctor` integration (13 already counts): keep behavior consistent — same label filter helper reused.
5. Never touch non-managed containers/images regardless of name (label filter only — test this).

## Acceptance Criteria

- [ ] itest: create 2 fixture rooms + 1 unmanaged busybox container → `clean` removes exactly the
  2 managed ones (unmanaged intact); with one marked protected, it survives.
- [ ] itest: `clean --all --yes` removes managed images; re-list shows zero managed objects.
- [ ] Non-TTY `clean --all` without `--yes` aborts, exit 1, nothing removed.
- [ ] Unit: report accounting; protection logic; label-filter exclusivity.

## Validation

`make itest` green; manual transcript of `clean` + `clean --all` in PR.

## Dependencies

15.

## Non-goals

Pruning by age, build-cache pruning (`docker builder prune` guidance goes in docs 37), automatic GC on victory (22 removes its own container explicitly).

## Design References

DESIGN §7.8, §11 F10.
