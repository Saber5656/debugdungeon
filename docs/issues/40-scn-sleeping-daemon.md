# Title

Scenario: sleeping-daemon (Floor 2 — crashlooping service)

## Summary

Floor-2 room: the heartbeat daemon crashloops because its config file contains an invalid key and
points its pidfile at a nonexistent directory; the player must read the crash log, fix the config,
and get the daemon stable.

## Context

Teaches: reading service logs, crashloop diagnosis, config validation flags, `ps` observation over
time. DESIGN §14 Floor 2 row 1. All Floor-2 rooms use the boot-script respawn-loop supervision
pattern (COOKBOOK §3–4) — no systemd.

## Scope

`scenarios/sleeping-daemon/` only.

## Detailed Requirements

1. `scenario.yaml`: id `sleeping-daemon`, floor 2, difficulty 2, topics `[services, logs]`,
   time_estimate_min 20. Locks:
   - `daemon-alive` / `The heart daemon beats steadily` — `heartd` process uptime > 10s
     (retry-with-deadline; manifest timeout 40).
   - `heartbeat-fresh` / `Fresh heartbeats reach the shrine` — `/var/lib/heartd/beat` mtime < 10s (retries).
2. Dockerfile:
   - `/usr/local/bin/heartd` (sh): on start, parses `/etc/heartd.conf` (KEY=VALUE lines; accepts
     `interval`, `pidfile`, `beatfile`); unknown key → `heartd: fatal: unknown directive '<key>' (line N)`
     to `/var/log/heartd.log`, exit 1; missing pidfile dir → fatal too. Healthy: writes pidfile,
     touches beatfile every `interval` seconds.
   - Broken `/etc/heartd.conf`: `interval=5`, `pidfil=/run/heartd-missing/heartd.pid` (typo key +
     bad dir — TWO layered faults discovered in sequence), `beatfile=/var/lib/heartd/beat`.
   - Boot script: respawn loop `while true; do heartd; sleep 2; done &` (crashloop visible in
     `ps`/log), boot-ok, sleep infinity.
   - `heartd --check /etc/heartd.conf` mode exists (validates without running) — the discoverable
     "config test" affordance (mentioned in `heartd --help`).
3. Checks: `alive.sh` — pid from pidfile exists AND `/proc/<pid>` older than 10s (use
   `cut -d. -f1 /proc/uptime` vs process start ticks, or simpler: two-sample check with sleep 5
   inside retry loop); MSG guides to `/var/log/heartd.log`. `beat.sh` — beatfile mtime fresh.
4. Hints: 01 `ps aux | grep heartd` keeps changing pid — something restarts and dies; where do
   daemons complain? (`/var/log/`). 02 read the fatal lines: an unknown directive AND a pidfile
   path — fix the typo (`pidfil`→`pidfile`) and make the directory exist or point somewhere sane.
   03 near-answer: correct conf lines + `mkdir -p /run/heartd`; then watch the log settle.
5. `solution.md`: crashloop triage narrative (observe → log → config test → fix → verify),
   Lesson Learned (crashloops, config validators, pidfiles).
6. `solution.sh`: `sed -i 's/^pidfil=/pidfile=/' /etc/heartd.conf; sed -i 's#/run/heartd-missing#/run/heartd#' /etc/heartd.conf; mkdir -p /run/heartd /var/lib/heartd` then wait ≤ 30s for beat.

## Acceptance Criteria

- [ ] Harness green both arches (pristine: both locks closed after boot).
- [ ] Fixing ONLY the typo leaves the daemon still dying (bad dir) — layered-fault sequencing verified in transcript.
- [ ] `ValidateDir` clean; image ≤ 150 MB; locks stable across 3 consecutive harness runs (no flake).
- [ ] Manual transcript of intended diagnosis path.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=sleeping-daemon`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

systemd units, logrotate, multi-service interplay (41+).

## Design References

DESIGN §14 Floor 2; COOKBOOK §3, §4, §6 (retry pattern).
