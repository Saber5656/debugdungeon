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
   - `imp-bound` / `The imp is bound` — spawner disabled (`/etc/cron.d/imp` absent/commented) AND
     count stable over 10s sample.
2. Dockerfile + boot (tmpfs populated at boot — same pattern note as 44):
   - `/usr/local/bin/imp` (sh): creates 200 zero-byte files per run in `/var/sessions/imp.d/`
     until IUse ≥ 95%, then stops (stability guard).
   - `/etc/cron.d/imp`: every minute, root, runs imp.
   - `/usr/local/bin/session-maker` (sh): `create` writes `/var/sessions/s-<ts>.json`; on ENOSPC
     prints `session-maker: cannot create session: No space left on device` (the misleading
     errno — inode exhaustion reports ENOSPC; the lesson's hook).
   - Boot: seed to ~95% IUse (do it in bulk with a seq loop — must finish < 20s with nr_inodes
     8192 → ~7.7k files, fast), start cron, boot-ok, sleep infinity.
3. Checks per above (POSIX `df -Pi` parsing; MSGs: `Space says fine; count the souls instead —
   df -i.` / spawner pointer).
4. Hints: 01 "no space" but `df -h` shows space?? disks run out of *two* things — check `df -i`.
   02 who makes numberless empty files? find the densest dir (`find /var/sessions -xdev | wc -l`,
   per-dir counts), and what schedules it. 03 `rm -rf /var/sessions/imp.d` (or find -delete),
   remove `/etc/cron.d/imp`; note why `rm imp.d/*` may fail (argv limit) — teach `find -delete`.
5. `solution.md` (+ Lesson: inode exhaustion, deceptive ENOSPC, argv limits on mass delete).
   `solution.sh`: remove cron entry, `find /var/sessions/imp.d -type f -delete; rmdir …`, verify probe.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed; boot seeding < 20s.
- [ ] `rm imp.d/*` failure mode reproduced and documented in solution.md (transcript evidence).
- [ ] nr_inodes schema path (SV019) exercised; `ValidateDir` clean; image ≤ 150 MB; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=inode-imp`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

ext4-specific inode tuning talk (tmpfs semantics only; solution.md may note the real-world analogue).

## Design References

DESIGN §6.2 (nr_inodes), §14 Floor 3; COOKBOOK §4, §13; issue 44 (boot-population pattern).
