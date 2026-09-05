# Title

Scenario: inode-imp (Floor 3 — inode exhaustion)

## Summary

Floor-3 room: plenty of free bytes, yet nothing can be created — an imp has exhausted the inodes
of the session store with millions of tiny files; the player must diagnose `df -i` and clean up
the right way.

## Context

Teaches: bytes vs inodes (`df -h` lies, `df -i` tells), finding file-count hotspots, deleting huge
file sets efficiently (`find -delete` vs `rm *` argv limits). DESIGN §14 Floor 3 row 2.

## Scope

`scenarios/inode-imp/` only.

## Detailed Requirements

1. `scenario.yaml`: id `inode-imp`, floor 3, difficulty 3, topics `[disk, filesystem]`,
   time_estimate_min 25; `mounts.tmpfs: [{path: /var/sessions, size_mb: 32, nr_inodes: 8192}]`. Locks:
   - `inodes-free` / `The session store can hold new souls` — `df -Pi /var/sessions` IUse ≤ 50%.
   - `sessions-work` / `New sessions are born` — app probe: `session-maker create` succeeds (writes a session file).
   - `imp-bound` / `The imp is bound` — exact predicate: no uncommented line in `/etc/cron.d/imp`
     invoking `/usr/local/bin/imp` (file absent is fine) AND file-count in `imp.d` stable across
     two samples 10s apart (timeout 30).
2. Dockerfile + boot (C3; tmpfs populated at boot — same pattern note as 44):
   - `/usr/local/bin/imp` (sh): creates zero-byte files in `/var/sessions/imp.d/` in a loop until
     `touch` fails (ENOSPC on inodes), then exits (stability guard = the filesystem itself).
   - `/etc/cron.d/imp`: every minute, root, runs imp (cron-file validity rules per 43).
   - `/usr/local/bin/session-maker` (sh): `create` writes `/var/sessions/s-<ts>.json`; on failure
     prints `session-maker: cannot create session: No space left on device` (the misleading
     errno — inode exhaustion reports ENOSPC; the lesson's hook).
   - Boot: `mkdir -p /var/sessions/imp.d`; seed by running `imp` once — it fills to actual inode
     ENOSPC (guarantees `sessions-work` is CLOSED on pristine; ~8k files, < 20s); start cron; C3
     boot-ok; `exec sleep infinity`.
3. Checks per above (POSIX `df -Pi` parsing; MSGs: `Space says fine; count the souls instead —
   df -i.` / spawner pointer).
4. Hints: 01 "no space" but `df -h` shows space?? disks run out of *two* things — check `df -i`.
   02 who makes numberless empty files? find the densest dir (`find /var/sessions -xdev | wc -l`,
   per-dir counts), and what schedules it. 03 remove `/etc/cron.d/imp`, then
   `find /var/sessions/imp.d -type f -delete && rmdir /var/sessions/imp.d` — and why `find
   -delete` is the pro habit over `rm imp.d/*` for huge file sets.
5. `solution.md` (+ Lesson: inode exhaustion, deceptive ENOSPC, mass-delete technique —
   `find -delete` taught as best practice with a NOTE about real-world argv limits; the limit
   itself is not reproduced at this file count, don't claim it is).
   `solution.sh`: remove cron entry, `find /var/sessions/imp.d -type f -delete; rmdir …`, verify probe.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed (inode ENOSPC proven by session-maker failing); boot seeding < 20s.
- [ ] nr_inodes schema path (SV019) exercised; `ValidateDir` clean; image ≤ 150 MB; no flake ×3.
- [ ] Deps note: tmpfs `nr_inodes` support comes from 07/15 (transitively required via 29) — verified present before starting.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=inode-imp`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

ext4-specific inode tuning talk (tmpfs semantics only; solution.md may note the real-world analogue).

## Design References

DESIGN §6.2 (nr_inodes), §14 Floor 3; COOKBOOK §4, §13; issue 44 (boot-population pattern).
