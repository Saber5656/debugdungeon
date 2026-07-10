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
   - `vault-breathes` / `The vault has room to breathe` — `/var/vault` usage ≤ 80%
     (parse `df -P /var/vault`; timeout 10). Pristine is ~100% (ENOSPC-seeded) → CLOSED.
   - `archivist-writes` / `The archivist can file records` — probe write
     `sh -c 'echo probe > /var/vault/records/.probe && rm /var/vault/records/.probe'` succeeds AND
     `/var/vault/records/.last-ok` mtime < 15s (the archivist touches it after each successful
     record; retry ≤ 25s, timeout 30). Pristine: ENOSPC → both fail → CLOSED.
   - `chatterbox-silenced` / `The chatterbox babbles no more` — flag file
     `/etc/chatterbox.enabled` ABSENT AND no `chatterbox` process running (`pgrep -f
     /usr/local/bin/chatterbox` empty; timeout 10). (Process-based, NOT growth-sampling — the
     size-guard makes growth an ambiguous signal; the review caught this.) Pristine: flag exists +
     process runs → CLOSED.
2. Dockerfile + boot (C3; boot owns ALL /var/vault content — tmpfs shadows the image, DESIGN §6.6
   exception; this is the pattern for 45 too):
   - Boot sequence: `mkdir -p /var/vault/records`; seed `/var/vault/debug.log` **to ENOSPC**
     (`dd if=/dev/zero of=/var/vault/debug.log bs=1M || true` — write until the tmpfs refuses;
     guarantees the archivist probe fails on pristine); start `archivist` loop
     (`/usr/local/bin/archivist`: every 5s writes a small record then touches
     `/var/vault/records/.last-ok`; ENOSPC failures logged to `/var/log/archivist.log`); start
     `chatterbox` loop (`/usr/local/bin/chatterbox`: while `/etc/chatterbox.enabled` exists,
     append noise-chunks to debug.log, tolerating ENOSPC with `|| true` + 2s sleep — alive but
     naturally blocked); C3 boot-ok marker; `exec sleep infinity`. Boot completes < 25s.
3. Check MSGs: `df` pointer / `The archivist still cannot file records.` /
   `Something still babbles into the vault — find the chatterbox and its enabling charm.`
4. Hints: 01 `df -h` → who's full? `du -a /var/vault | sort -n | tail`. 02 the log is huge —
   find its writer (`ps`, `ls /etc/*.enabled`), silence it FIRST (flag + process), then reclaim.
   03 `rm /etc/chatterbox.enabled; pkill -f /usr/local/bin/chatterbox; : > /var/vault/debug.log`
   (truncate beats rm for open handles — explain).
5. `solution.md` (+ Lesson: full disks break writes; find growth sources; truncation vs deletion
   with open fds; silence-then-clean ordering). `solution.sh`: remove flag, pkill chatterbox,
   truncate log, wait ≤ 20s for `.last-ok` freshness.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed (ENOSPC seed proven by the probe write failing); boot completes < 25s.
- [ ] Truncate-only route (flag/process untouched) leaves `chatterbox-silenced` closed; flag-removed-but-process-alive also closed — trap matrix verified.
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
