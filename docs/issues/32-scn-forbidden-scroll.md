# Title

Scenario: forbidden-scroll (Floor 1 — permissions and least privilege)

## Summary

Build the Floor-1 room where a service user cannot read its config because of botched permissions;
the fix must be least-privilege (world-readable is rejected by a dedicated lock).

## Context

Teaches: `ls -l`, ownership vs mode, directory execute bits, groups, and *thoughtful* permission
repair (not `chmod 777`). DESIGN §14 Floor 1 row 3.

## Scope

`scenarios/forbidden-scroll/` only.

## Detailed Requirements

1. `scenario.yaml`: id `forbidden-scroll`, floor 1, difficulty 1, topics `[permissions, users]`,
   time_estimate_min 20. Locks:
   - `scribe-reads` / `The scribe can read the scroll`
   - `not-world-readable` / `The scroll is not left open to all`
   - `heartbeat-fresh` / `The scriptorium breathes`
2. Dockerfile:
   - Create user `scribe` (system user, home `/home/scribe`).
   - `/etc/scroll/config.yaml` (content: a few yaml keys incl `phrase: lumen-in-tenebris`),
     owner `root:root`, mode `0600`; dir `/etc/scroll` mode `0700 root:root` (double fault).
   - Service `/usr/local/bin/scrolld` (sh): loop as scribe — reads the config's `phrase`, writes it
     + timestamp to `/var/run/scroll/heartbeat` every 5s; `/var/run/scroll` owned `scribe`, 0755.
   - Boot script: start `su -s /bin/sh scribe -c scrolld &` style loop (respawn wrapper), boot-ok, sleep infinity.
     While perms are broken, scrolld logs `permission denied` to `/var/log/scrolld.log` (visible symptom trail).
3. Checks:
   - `scribe-reads.sh`: `su -s /bin/sh scribe -c 'cat /etc/scroll/config.yaml' >/dev/null 2>&1`
     else `MSG: The scribe still cannot read the scroll (permission denied).`
   - `not-world-readable.sh`: fail if file has `o+r` OR dir has `o+x`:
     `MSG: The scroll lies open for any passerby — seal it from others.` (stat -c %a parsing; POSIX-safe arithmetic).
   - `heartbeat-fresh.sh`: retry-with-deadline pattern (cookbook §6): up to 20s for heartbeat mtime
     < 15s old AND containing the phrase; `MSG: No fresh heartbeat from the scriptorium.` timeout 30 in manifest.
4. Hints: 01 whoami/`ls -l` the config and its directory — who may read, and can scribe even
   *enter* the dir? 02 group-based least privilege: make a shared group or chown group to scribe;
   dir needs `x` for traversal. 03 near-answer: `chgrp scribe file+dir; chmod 640 file; chmod 750 dir`.
5. `solution.md`: full diagnosis narrative (service log → su test → ls -l chain), the least-priv
   fix, why 777/755+o+r fails the third lock; Lesson Learned (traversal bits, group ownership, least privilege).
6. `solution.sh`: `chgrp scribe /etc/scroll /etc/scroll/config.yaml; chmod 750 /etc/scroll; chmod 640 /etc/scroll/config.yaml`
   (heartbeat lock then passes once the loop cycles — harness lock timeout accommodates via retry check).

## Acceptance Criteria

- [ ] Harness green both arches; pristine state: `scribe-reads` and `heartbeat-fresh` closed, `not-world-readable` OPEN (pristine 0600 isn't world-readable — ≥1 closed satisfies ADR-006 gate; note this asymmetry in PR).
- [ ] `chmod 777` route: first two locks open, third closed with its MSG — transcript proves the teaching moment.
- [ ] `ValidateDir` clean; image ≤ 150 MB; heartbeat check passes within one retry window after fix.
- [ ] Manual transcript: diagnose via `/var/log/scrolld.log`, `su` probe, fix, escape.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=forbidden-scroll`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

ACLs/setfacl (not installed), SELinux, sudo puzzles (NNP — cookbook §5).

## Design References

DESIGN §14 Floor 1, §6.3 (retry pattern), §7.4 (why root entry + su works: SETUID/SETGID caps present); COOKBOOK §4/§6.
