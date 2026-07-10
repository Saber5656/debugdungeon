# Title

Scenario: zombie-horde (Floor 2 — runaway process spawner)

## Summary

Floor-2 room: a cron-driven necromancer script multiplies worker processes without bound (safely
capped by the container pids limit); the player must stop the source, clear the horde, and keep
the legitimate worker running.

## Context

Teaches: process-tree reading (`ps -ef --forest`), cron as a spawn source, safe mass-kill
(`pkill -f`), distinguishing legit vs runaway processes. DESIGN §14 Floor 2 row 3. The room
deliberately exercises our resource caps (DESIGN §7.4) as a player-visible feature.

## Scope

`scenarios/zombie-horde/` only.

## Detailed Requirements

1. `scenario.yaml` (C3): id `zombie-horde`, floor 2, difficulty 2, topics `[processes, cron]`,
   time_estimate_min 20; resources: pids 256 (default). Dockerfile (C3 preamble) installs `cron`,
   `procps`, `psmisc` (apt rules per C3/COOKBOOK §10).
   Locks:
   - `horde-cleared` / `The horde is dust` — `pgrep -c -f zombie-worker` == 0 (after grace; retries).
   - `necromancer-stopped` / `The summoning circle is broken` — `timeout_sec: 60`; check logic
     (budget ≤ 55s): assert no uncommented `/etc/cron.d/necromancer` line invoking zombie-worker
     (file may be absent), then two samples of `pgrep -c -f zombie-worker` taken 10s apart within
     the budget, both == 0. Sufficiency argument (documented in the check header): cron has
     minute granularity, so entry-absent + count-zero-sustained-10s proves the horde is dead and
     unsummonable — no 70s wait needed.
   - `gravekeeper-alive` / `The gravekeeper still works` — `/var/lib/grave/ok` mtime < 15s (retry
     ≤ 25s, timeout 30) — the DESIGN §14 "still works" lock is freshness-based, not pgrep-based
     (catches both killed and wedged gravekeepers).
2. Dockerfile:
   - `/usr/local/bin/zombie-worker` (sh): sleep-loop that also spawns a sibling every 30s
     (multiplication bounded naturally by pids limit; each worker checks `pgrep -c -f zombie-worker`
     and refrains above 60 — keeps the room responsive while still "a horde").
   - `/etc/cron.d/necromancer`: `* * * * * root /usr/local/bin/zombie-worker >/dev/null 2>&1` (the source).
   - `/usr/local/bin/grave-worker` (sh): legit maintenance loop touching `/var/lib/grave/ok`
     every 10s.
   - Boot script (exact behavior): start respawn loop
     `while true; do /usr/local/bin/grave-worker; sleep 3; done &`; start `cron`; seed 5 initial
     zombies (`for i in 1 2 3 4 5; do /usr/local/bin/zombie-worker & done`); C3 boot-ok marker;
     `exec sleep infinity`.
3. Check MSGs guide: `The circle still glows in /etc/cron.d/…` /
   `Shambling things remain — count them with pgrep.` / `No fresh sign of the gravekeeper's work.`
4. Hints: 01 `ps -ef --forest`, who keeps making these? things that run "every minute" live where?
   02 `/etc/cron.d/necromancer` — remove/comment it, then clear survivors with `pkill -f zombie-worker`.
   03 near-answer incl. warning: don't pkill the gravekeeper (`grave-worker` ≠ `zombie-worker`).
5. `solution.md` (+ Lesson: kill the source first, then the symptoms; cron spawn paths; careful
   pkill patterns). `solution.sh`: `rm -f /etc/cron.d/necromancer; pkill -f zombie-worker || true;`
   verify loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: first two locks closed, gravekeeper OPEN.
- [ ] Overeager `pkill -f worker` kills gravekeeper → third lock closes with its MSG (transcript proves the trap).
- [ ] Zombie population bounded: scripted sampling (`for i in $(seq 6); do pgrep -c -f zombie-worker; sleep 20; done` on a pristine room) shows counts < 70 throughout — transcript attached.
- [ ] All lock timeouts ≤ 60 (SV014-compliant); no flake ×3 runs.
- [ ] `ValidateDir` clean; image ≤ 200 MB.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=zombie-horde`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

True fork-bombs (hostile to the harness), cgroup lessons (v2 idea).

## Design References

DESIGN §14 Floor 2, §7.4 (pids clamp as a feature); COOKBOOK §4, §6; issue 08 SV014.
