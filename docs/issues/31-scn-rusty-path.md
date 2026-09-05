# Title

Scenario: rusty-path (Floor 1 — PATH sabotage)

## Summary

Build the Floor-1 room where a corrupted login `PATH` (missing `/usr/local/bin`, poisoned by a
decoy directory) hides the gate-opening binary.

## Context

Teaches: how PATH resolution works, `command -v`/`which`, login vs non-login shells, reading
`/etc/profile`. DESIGN §14 Floor 1 row 2; the DESIGN §3.2 transcript is this room.

## Scope

`scenarios/rusty-path/` only.

## Detailed Requirements

1. `scenario.yaml`: id `rusty-path`, floor 1, difficulty 1, topics `[shell, path]`,
   time_estimate_min 15, defaults elsewhere. Lore: the PATH sigils on the walls are rusted; the
   gate lever `open-gate` exists but the walls no longer find it; a decoy lever crumbles at a touch.
   Locks:
   - `path-restored` / `The PATH sigil is restored` / `checks/path.sh`
   - `gate-opened` / `The gate stands open` / `checks/gate.sh`
2. Dockerfile (one breakage layer, `ARG BASE` digest pattern):
   - Real binary `/usr/local/bin/open-gate` (sh script, 0755): writes `opened at $(date)` to
     `/var/dungeon/gate-opened` and prints a victory-ish line.
   - Decoy `/opt/decoy/open-gate` (0755): prints `The lever crumbles to rust in your hand.`,
     exits 1, and does NOT create the marker.
   - Breakage: write `/etc/profile.d/00-rusty-sigils.sh` containing
     `PATH=/opt/decoy:/usr/bin:/bin:/usr/sbin:/sbin; export PATH` (note: `/usr/local/bin` absent,
     decoy first). **Placement rationale (load-bearing):** profile.d scripts run in lexical order,
     so `00-` runs BEFORE the engine's `zz-debugdungeon.sh` (18) — the engine re-prepends
     `/dungeon/bin`, keeping `escape`/`hint` reachable while the sabotage still hides
     `/usr/local/bin`. Never append PATH sabotage after the profile.d loop in `/etc/profile`
     itself (it would run after `zz-` and break the helpers). Do not modify `/root/.bashrc`.
   - Boot script standard (C3 boot contract). `/var/dungeon/` created.
3. `checks/path.sh` — must evaluate the **login-shell** PATH, not the exec default:
   ```sh
   resolved=$(su -l root -c 'command -v open-gate' 2>/dev/null)
   if [ "$resolved" != "/usr/local/bin/open-gate" ]; then
     echo "MSG: A login shell still cannot find the true open-gate (found: ${resolved:-nothing})."
     exit 1
   fi
   exit 0
   ```
   (`su -l` re-reads `/etc/profile`; engine helpers prepend `/dungeon/bin` via profile.d — harmless,
   `open-gate` isn't there. `su` exists in bookworm-slim; verify at build, else install `util-linux` login tools.)
4. `checks/gate.sh`: `/var/dungeon/gate-opened` exists → 0; else
   `MSG: The gate has not been commanded to open — find and run open-gate.`
5. Hints:
   - 01: Where does a shell look for commands? Compare `echo $PATH` with `command -v open-gate`;
     something rusty lurks where login shells read their sigils (`/etc/profile.d/`)…
   - 02: Two problems: a decoy dir comes first, and `/usr/local/bin` is missing entirely. Fix the
     sigil file's PATH line, then start a login shell (`bash -l`) to pick it up.
   - 03: near-answer: edit `/etc/profile.d/00-rusty-sigils.sh` so PATH starts with
     `/usr/local/bin` and drops `/opt/decoy` (or delete the whole sigil file — the default PATH
     already includes /usr/local/bin), then run `open-gate`.
6. `solution.md`: symptom → diagnosis chain (`command not found` → `echo $PATH` → decoy discovery
   → `/etc/profile`) → fix → Lesson Learned (PATH order, decoys/shadowing, login-shell config files).
7. `solution.sh` (idempotent):
   ```sh
   rm -f /etc/profile.d/00-rusty-sigils.sh
   su -l root -c 'open-gate'
   ```
   (Deleting the sabotage file restores Debian's stock login PATH, which includes
   `/usr/local/bin`; editing the file to a sane PATH is the equally-valid player route.)
8. Alternate player routes and the lock's arbitration: any state where a LOGIN shell resolves
   `command -v open-gate` to `/usr/local/bin/open-gate` passes `path-restored` — editing the sigil
   file, deleting it, or adding an earlier correct export all work. Merely running the binary by
   absolute path does NOT (lock 1 stays closed, lock 2 opens — partial-fix teaching moment).
   Note the stock-PATH assumption: verify at implementation that bookworm's default login PATH
   (from `/etc/profile`) includes `/usr/local/bin` — it does on stock Debian; if the base image
   differs, adjust the breakage to also strip it there and update solution.sh accordingly.

## Acceptance Criteria

- [ ] `ValidateDir` clean; harness (29) green both arches (pristine: both locks closed).
- [ ] Alt-route evidence: deleting `/etc/profile.d/00-rusty-sigils.sh` AND editing it to a corrected PATH both pass `path-restored`; deleting only `/opt/decoy` does NOT (PATH still lacks /usr/local/bin) — closed-lock transcript included.
- [ ] Running `open-gate` via absolute path WITHOUT fixing PATH leaves `path-restored` closed (partial-fix teaches the lock model) — verified in transcript.
- [ ] Image ≤ 150 MB; hints escalate per cookbook §8.
- [ ] Manual transcript incl. the failed-then-fixed flow.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=rusty-path`; transcript in PR.

## Dependencies

12, 29.

## Non-goals

zsh/fish profile variants; non-root entry.

## Design References

DESIGN §14 Floor 1, §3.2 transcript, §6.3 (login-shell check nuance); COOKBOOK §4/§6.
