# Title

Scenario image ensure/build pipeline

## Summary

Implement `dockerx.EnsureImage`: content-hash-tagged build of a scenario's image from the
materialized embedded context, with streamed logs, friendly failure classification, and
cancellation.

## Context

DESIGN §7.2 defines the cache key discipline (ADR-004). First-build UX and offline behavior
(F3/F4) are shaped here.

## Scope

- `internal/dockerx/build.go` + tests
- Not: registry pushes (never), multi-platform cross-builds (daemon-native only)

## Detailed Requirements

1. `EnsureImage(ctx, api, reg, l *scenario.Loaded, progress io.Writer) (ref string, built bool, err error)`:
   - `ref = "debugdungeon/scn-" + l.Spec.ID + ":" + scenario.Short(l.Hash)`.
   - `ImageList` filtered by reference `ref`; hit → return `(ref, false, nil)`.
   - Miss → `reg.Materialize` (10) → tar the build context dir in-memory/stream (no temp tar file;
     `archive/tar` walking the dir in sorted relative-path order; regular files + dirs only —
     symlinks already impossible post-Materialize; headers: uid/gid 0, empty uname/gname, mode
     0644 files / 0755 dirs, zero mtime, USTAR/PAX default) → `ImageBuild` with:
     `Tags=[ref]`, `Dockerfile="Dockerfile"`, `Remove=true`, `ForceRemove=true`, `PullParent=false`,
     `Labels={"com.debugdungeon.managed":"true","com.debugdungeon.scenario":l.Spec.ID,"com.debugdungeon.version":<cli version>}`,
     platform empty (daemon native).
   - **Caller contract (TB4):** bundled/external-authoring content only in MVP; when packs exist
     (Wave 8), callers must have passed the §10.6 trust gate BEFORE calling `EnsureImage` — noted
     in the function doc comment; enforced by 61's gate ordering.
   - Decode the JSON message stream (`jsonmessage.JSONMessage`): `stream` field lines → debug log
     + ring buffer + (step lines matching `^Step |^ ---> `) to `progress`; `status` lines (pulls)
     → debug log only; `error`/`errorDetail` → terminates with the classification below; malformed
     JSON lines → logged raw, ignored; EOF without error message = success. Everything written to
     `progress` or later shown from the ring buffer passes `textsafe.Sanitize` (Dockerfile-
     controlled bytes, TB3).
   - Error stream message → classify: contains pull/DNS/TLS/timeout patterns
     (`"no such host"`, `"timeout"`, `"TLS handshake"`, `"failed to resolve"`, `"pull access denied"`)
     → `ErrBuildNetwork` (F3 message: first build needs the network to fetch the pinned base image);
     otherwise `ErrBuildFailed` carrying the ring buffer (F4: print last 30 lines + bug-report pointer for bundled scenarios).
2. Cancellation: ctx cancel must abort the build request, close the `ImageBuild` response body,
   and return a context-classed error promptly (Ctrl-C during first build).
3. `built==true` lets callers print "Forging this room for the first time…".
4. Determinism note: base image is pinned by digest in the Dockerfile (cookbook); `PullParent=false`
   ensures we never silently float; the daemon auto-pulls a missing base by digest.

## Acceptance Criteria

- [ ] Unit: tag computation; ring buffer keeps exactly last 30; classification table-tested against fixture error strings.
- [ ] Unit: tar stream is deterministic (two runs → identical bytes) and contains only the build context files.
- [ ] itest (real daemon): build a dedicated fixture scenario (constructed as a `scenario.Loaded` from `internal/dockerx/testdata/buildscn` — NOT `_template`, which the registry ignores) twice — first `built=true`, second `built=false` (cache hit); image carries all three labels.
- [ ] itest: broken Dockerfile fixture → `ErrBuildFailed` and buffer includes the failing step.
- [ ] Ctrl-C (context cancel) during itest build aborts < 3s.

## Validation

`make test` + `make itest` green; PR includes a build transcript showing the 30-line failure tail rendering.

## Dependencies

09, 10, 11, 13.

## Non-goals

BuildKit-specific features, cross-arch emulation, image pruning (16), progress UI polish (21).

## Design References

DESIGN §7.2, §11 (F3, F4); ADR-004.
