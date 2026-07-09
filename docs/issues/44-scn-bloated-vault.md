# Title

Scenario: bloated-vault (Floor 3 — disk full)

## Summary

Floor-3 room: the vault (a 64 MB tmpfs) is packed solid by a runaway debug log, so the archivist
app can't write; the player must find the space hog, stop its source, and reclaim space.

## Context

Teaches: `df -h`, `du -sh` drill-down, biggest-file hunting, log-spam sources, truncate-vs-delete
for open file handles. DESIGN §14 Floor 3 row 1. First scenario exercising spec tmpfs mounts
(07/15 `mounts.tmpfs`) — the ONLY sanctioned place for disk-full puzzles (COOKBOOK §4/§13).

## Scope

`scenarios/bloated-vault/` only.

## Detailed Requirements

1. `scenario.yaml`: id `bloated-vault`, floor 3, difficulty 3, topics `[disk, logs, filesystem]`,
   time_estimate_min 25; `mounts.tmpfs: [{path: /var/vault, size_mb: 64}]`. Locks:
   - `vault-breathes` / `The vault has room to breathe` — `/var/vault` usage ≤ 80% (parse `df -P /var/vault`).
   - `archivist-writes` / `The archivist can file records` — `su`-less: exec test
     `sh -c 'echo probe > /var/vault/records/.probe && rm /var/vault/records/.probe'` succeeds AND
     archivist's own status file shows a recent successful write (retries).
   - `chatterbox-silenced` / `The chatterbox babbles no more` — the debug-spam loop disabled:
     `/var/vault/debug.log` growth == 0 over a 10s sample (two-stat compare within the check).
2. Dockerfile + boot:
   - Boot script starts: `archivist` loop (writes a small record to `/var/vault/records/` every 5s,
     logs ENOSPC failures to `/var/log/archivist.log`) and `chatterbox` loop (appends 1 MB/s of
     noise to `/var/vault/debug.log` while flag file `/etc/chatterbox.enabled` exists — the
     disable affordance) — **spam runs at boot until the tmpfs is ~full, then the writer naturally
     blocks on ENOSPC**: cap chatterbox with a size guard (stop appending at 62 MB) so the room is
     stable, not thrashing. Seed the tmpfs near-full at boot (dd from /dev/zero into debug.log until 62 MB).
   - Note: tmpfs content cannot be baked into the image (mounted at run time) — ALL /var/vault
     population happens in the boot script (document prominently; this is the pattern for 45 too).
   - Boot-ok only after seeding completes.
3. Checks per above; MSGs: `df` pointer / `Something still floods the vault — watch the log grow.`
4. Hints: 01 `df -h` → who's full? `du -a /var/vault | sort -n | tail`. 02 the log is huge and
   still growing — find its writer (`ls /etc/*.enabled`, `ps`), disable it, then reclaim.
   03 `rm /etc/chatterbox.enabled; : > /var/vault/debug.log` (truncate beats rm for open handles — explain).
5. `solution.md` (+ Lesson: full disks break writes; find growth sources; truncation vs deletion
   with open fds). `solution.sh`: disable flag, truncate log, wait for archivist success.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed; boot (incl. 62 MB seed) completes < 25s.
- [ ] rm-only route (delete flag file NOT removed) leaves `chatterbox-silenced` closed — trap verified.
- [ ] Host safety: writes confined to the tmpfs (inspect: no disk growth in container layer beyond noise) — reviewer checkbox.
- [ ] `ValidateDir` clean (tmpfs schema path); image ≤ 150 MB; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=bloated-vault`; transcripts in PR.

## Dependencies

12, 29 (tmpfs support from 07/15 already in MVP).

## Non-goals

Quota tooling, logrotate configuration (mentioned in solution.md as the real-world fix).

## Design References

DESIGN §6.2 (mounts.tmpfs), §14 Floor 3; COOKBOOK §4, §13.
