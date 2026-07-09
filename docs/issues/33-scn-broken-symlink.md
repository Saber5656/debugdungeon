# Title

Scenario: broken-symlink (Floor 1 — dangling release links)

## Summary

Build the Floor-1 room where the app's `current` symlink dangles after a botched deploy; the
player must relink to the correct (non-corrupt) release.

## Context

Teaches: `readlink`, `ls -l` on links, atomic `ln -sfn`, and "don't trust the newest dir".
DESIGN §14 Floor 1 row 4. Note: scenario CONTENT contains symlinks *inside the image via
Dockerfile RUN* — the scenario source tree itself stays symlink-free (SV021); links are created at
build time.

## Scope

`scenarios/broken-symlink/` only.

## Detailed Requirements

1. `scenario.yaml`: id `broken-symlink`, floor 1, difficulty 1, topics `[filesystem, deploy]`,
   time_estimate_min 15. Locks:
   - `link-resolves` / `The signpost points somewhere real`
   - `not-corrupt` / `It points to a sound release`
   - `app-ok` / `The gatekeeper app answers`
2. Dockerfile (breakage layer):
   - `/opt/app/releases/v1.2.3/bin/run` (sh, 0755): prints `ROOM_OK v1.2.3`.
   - `/opt/app/releases/v1.3.0/bin/run`: prints garbage and exits 1; plus marker file
     `/opt/app/releases/v1.3.0/CORRUPT` (content: `deploy interrupted 03:12`).
   - `RUN ln -s /opt/app/releases/v1.3.1 /etc/app/current` (dangling — v1.3.1 never existed).
   - `/usr/local/bin/appctl` (sh): `status` subcommand executes `/etc/app/current/bin/run`;
     failure prints the exec error (symptom trail).
   - Deploy log `/var/log/deploy.log` narrating the interrupted upgrade (breadcrumb).
   - Standard boot script.
3. Checks:
   - `link.sh`: `[ -e /etc/app/current ]` (dereferences) else `MSG: /etc/app/current still points into the void (readlink it).`
   - `corrupt.sh`: `[ ! -e /etc/app/current/CORRUPT ]` else `MSG: That release bears the mark of a broken deploy.`
   - `app.sh`: `/usr/local/bin/appctl status 2>/dev/null | grep -q '^ROOM_OK'` else `MSG: The gatekeeper app still fails to speak.`
4. Hints: 01 `appctl status` fails — what does `/etc/app/current` actually point at (`ls -l`,
   `readlink`)? 02 list `/opt/app/releases` — newest isn't healthiest; check for leftover markers.
   03 `ln -sfn /opt/app/releases/v1.2.3 /etc/app/current`.
5. `solution.md`: diagnosis chain (appctl error → readlink → releases listing → CORRUPT marker →
   deploy log), atomic relink with `ln -sfn`, Lesson Learned (dangling links; `-e` vs `-L` tests;
   atomic symlink swaps in deploys).
6. `solution.sh`: `ln -sfn /opt/app/releases/v1.2.3 /etc/app/current`.
7. Trap coverage: player linking to v1.3.0 opens `link-resolves` but not `not-corrupt`/`app-ok` —
   the lock MSGs guide onward.

## Acceptance Criteria

- [ ] Harness green both arches (pristine: all three locks closed).
- [ ] Linking to v1.3.0 leaves exactly `not-corrupt` + `app-ok` closed (transcript).
- [ ] `ValidateDir` clean (source tree has no symlinks — links made in RUN); image ≤ 150 MB.
- [ ] Manual transcript of intended path.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=broken-symlink`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Multi-hop link chains, hardlink lessons.

## Design References

DESIGN §14 Floor 1; issue 08 SV021 (why links live in the Dockerfile); COOKBOOK §4.
