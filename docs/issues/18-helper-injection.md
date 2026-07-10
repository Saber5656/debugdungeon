# Title

In-container helper injection: escape/hint/giveup, banner, PATH wiring

## Summary

Inject the game helpers and room banner into a created (not yet started) container via a tar
`CopyToContainer`, per DESIGN §7.6.

## Context

Helpers turn the sentinel protocol (ADR-003) into player-visible commands. Injection at create
time keeps scenario images clean (no engine files in content, DESIGN §6.6 spoiler hygiene).

## Scope

- `internal/dockerx/inject.go` + tests
- Not: banner text composition (21 builds it via ui; this issue takes it as an argument)

## Detailed Requirements

1. `InjectHelpers(ctx, api, containerID string, motd []byte) error` — builds an in-memory tar and
   `CopyToContainer(ctx, containerID, "/", tar, container.CopyToContainerOptions{})` (add this
   method to `dockerx.API` per CONVENTIONS C2). Tar layout: RELATIVE header names with explicit
   directory entries `dungeon/` 0755, `dungeon/bin/` 0755, `etc/profile.d/` 0755, then files
   (all uid/gid 0, mtime zeroed):
   - `dungeon/bin/escape` 0755: `#!/bin/sh\nexit 42\n`
   - `dungeon/bin/hint` 0755: `#!/bin/sh\nexit 43\n`
   - `dungeon/bin/giveup` 0755: `#!/bin/sh\nexit 44\n`
   - `dungeon/.motd` 0644 (DESIGN §7.6 path): `textsafe.Sanitize(string(motd), 4096)` bytes —
     caller passes already-sanitized content (11); the defensive re-sanitize here is intended
     chokepoint duplication.
   - `etc/profile.d/zz-debugdungeon.sh` 0644:
     ```sh
     PATH="/dungeon/bin:$PATH"; export PATH
     if [ -n "$DEBUGDUNGEON" ] && [ -n "$DD_REENTRY" ] && [ -f /dungeon/.motd ]; then
       cat /dungeon/.motd
     fi
     ```
     Banner de-duplication contract (with issue 21): the HOST prints the full banner exactly once
     per `play` invocation before the first shell; re-entry exec sessions (after hint/failed
     escape) set `DD_REENTRY=1` in the exec Env, so the in-container motd prints then and only then.
2. Ordering contract: called after `CreateRoom`, before `StartRoom` (works on created containers;
   test this — no exec needed).
3. Failure handling: any error → typed `ErrInjectFailed` (play aborts and removes the container; wiring in 21).
4. Robustness notes (document in code): helpers are reachable even when the scenario sabotages
   PATH, via absolute `/dungeon/bin/escape` — the host-side banner (21) always mentions absolute
   paths. `profile.d` only affects login shells of bash/ash-compatible shells; entry.shell default
   is bash `-l` (17). Cookbook forbids scenarios touching `/dungeon` (12).
5. No other files, no scenario-specific logic here.
6. Forward-compat note: issue 65 (ambience) later widens this signature to
   `InjectHelpers(ctx, api, containerID, InjectOpts{Motd, RoomID, Color})` to add PS1 theming.
   v1 MVP ships the `motd []byte` form above; keep the profile.d snippet in one easily-extended
   place so 65's change is additive.

## Acceptance Criteria

- [ ] Unit: tar layout golden test (relative paths, dir entries, modes, contents byte-exact; motd sanitization applied).
- [ ] itest: create room → inject → start → `exec cat /dungeon/.motd` matches; `exec /bin/bash -lc 'command -v escape'` → `/dungeon/bin/escape`; non-TTY exec of `/bin/bash -lc escape` inspects exit code 42.
- [ ] itest: `bash -l` WITHOUT `DD_REENTRY` prints nothing extra; with `DD_REENTRY=1` in Env it prints the motd (banner-dedup contract).
- [ ] Injection into a created-but-not-started container succeeds.

## Validation

`make test` + `make itest` green.

## Dependencies

11, 15.

## Non-goals

Per-floor themed motd art (65), PS1 theming (65), hint delivery content (23).

## Design References

DESIGN §7.6, §3.2, §10.5; ADR-003.
