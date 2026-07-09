# Title

In-container helper injection: escape/hint/giveup, banner, PATH and history wiring

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
   `CopyToContainer(containerID, "/", tar)`. Entries (all uid/gid 0, mtime zeroed):
   - `/dungeon/bin/escape` 0755: `#!/bin/sh\nexit 42\n`
   - `/dungeon/bin/hint` 0755: `#!/bin/sh\nexit 43\n`
   - `/dungeon/bin/giveup` 0755: `#!/bin/sh\nexit 44\n`
   - `/dungeon/motd` 0644: the `motd` bytes — caller MUST pass already-sanitized content (11);
     defensive `textsafe.Sanitize` applied here too (chokepoint duplication is intended).
   - `/etc/profile.d/zz-debugdungeon.sh` 0644:
     ```sh
     PATH="/dungeon/bin:$PATH"; export PATH
     if [ -n "$DEBUGDUNGEON" ] && [ -z "$DD_MOTD_SHOWN" ] && [ -f /dungeon/motd ]; then
       cat /dungeon/motd
       DD_MOTD_SHOWN=1; export DD_MOTD_SHOWN
     fi
     ```
2. Ordering contract: called after `CreateRoom`, before `StartRoom` (works on created containers;
   test this — no exec needed).
3. Failure handling: any error → typed `ErrInjectFailed` (play aborts and removes the container; wiring in 21).
4. Robustness notes (document in code): helpers are reachable even when the scenario sabotages
   PATH, via absolute `/dungeon/bin/escape` — the host-side banner (21) always mentions absolute
   paths. `profile.d` only affects login shells of bash/ash-compatible shells; entry.shell default
   is bash `-l` (17). Cookbook forbids scenarios touching `/dungeon` (12).
5. No other files, no scenario-specific logic here.

## Acceptance Criteria

- [ ] Unit: tar layout golden test (paths, modes, contents byte-exact; motd sanitization applied).
- [ ] itest: create room → inject → start → `exec cat /dungeon/motd` matches; `exec sh -lc 'command -v escape'` → `/dungeon/bin/escape`; `exec sh -lc 'escape'; echo $?` shell exits 42 (via non-TTY exec of `sh -lc "escape"` and inspecting exit code).
- [ ] itest: second login shell in same container does not re-print motd (DD_MOTD_SHOWN export) — single-session scope acceptable; assert via `sh -lc 'true'` output empty when var pre-set.
- [ ] Injection into a created-but-not-started container succeeds.

## Validation

`make test` + `make itest` green.

## Dependencies

11, 15.

## Non-goals

Per-floor themed motd art (65), PS1 theming (65), hint delivery content (23).

## Design References

DESIGN §7.6, §3.2, §10.5; ADR-003.
