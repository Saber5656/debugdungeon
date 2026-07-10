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

1. `ListManaged(ctx, api) (Managed, error)` where
   `Managed{Containers []ManagedContainer{ID, Name, ScenarioID, State string}; Images []ManagedImage{ID, Ref string, SizeBytes int64}}`
   — filtered by label `com.debugdungeon.managed=true`; containers all states; stable sort by
   (ScenarioID, ID) / (Ref, ID); image sizes deduped by image ID before summation.
2. `Clean(ctx, api, protected Protected, all bool) (Report, error)` where
   `Protected{ContainerID, ImageRef string}` (both may be empty):
   - Containers whose FULL ID ≠ protected: stop (5s) if running, remove force. Protected listed as
     "kept (active run)". Stale protected IDs (no such container) are ignored with a debug log.
   - `all`: additionally remove managed images EXCEPT `protected.ImageRef` (`ImageRemove` force,
     prune children) — the active run's image must survive for `reset` (DESIGN §7.8).
   - `Report{RemovedContainers, KeptContainers []string; RemovedImages int; FreedBytes int64}`.
   - Confirmation/TTY policy lives in the CLI layer (below), NOT in this function.
3. `clean` command wiring:
   - Protected set: read the active run via `game.ActiveRunRefs(paths) (containerID, imageRef string, ok bool)`
     — a minimal helper INTRODUCED HERE in `internal/game/active.go` reading `run.json` with a
     3-field anonymous struct decode (missing/corrupt file → `ok=false`, never an error surface).
     Issue 20 replaces its internals with the real store, keeping the signature (code comment).
   - Flags: `--all`, `--yes`. `--all` confirmation prompt shows count + total size; non-TTY without
     `--yes` → abort exit 1. `clean` NEVER touches `run.json` (DESIGN §7.8).
   - Output: table of actions + `freed ~X.Y GB` for images.
4. `doctor` integration (13 already counts): keep behavior consistent — same label filter helper reused.
5. Never touch non-managed containers/images regardless of name (label filter only — test this).

## Acceptance Criteria

- [ ] itest (fixtures created directly via the Docker API with our labels + a plain alpine/busybox
  container — no registry/build dependency): `clean` removes exactly the 2 managed ones (unmanaged
  intact); with one protected, it survives.
- [ ] itest: `clean --all --yes` removes managed images EXCEPT a protected image ref; re-list shows only it.
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
