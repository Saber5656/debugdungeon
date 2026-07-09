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

1. `scenario.yaml`: id `zombie-horde`, floor 2, difficulty 2, topics `[processes, cron]`,
   time_estimate_min 20; resources: pids 256 (default). Install `cron`, `procps`, `psmisc`.
   Locks:
   - `horde-cleared` / `The horde is dust` — `pgrep -c -f zombie-worker` == 0 (after grace; retries).
   - `necromancer-stopped` / `The summoning circle is broken` — the cron entry is gone/commented
     AND no new zombie appears within 70s of check start (two-phase check: count, sleep-in-retries, recount==0).
   - `gravekeeper-alive` / `The gravekeeper still works` — legit `gravekeeper` process running (it
     must NOT be caught by careless `pkill -f zombie`... its cmdline is distinct but similar: `grave-worker`).
2. Dockerfile:
   - `/usr/local/bin/zombie-worker` (sh): sleep-loop that also spawns a sibling every 30s
     (multiplication bounded naturally by pids limit; each worker checks `pgrep -c -f zombie-worker`
     and refrains above 60 — keeps the room responsive while still "a horde").
   - `/etc/cron.d/necromancer`: `* * * * * root /usr/local/bin/zombie-worker >/dev/null 2>&1` (the source).
   - `/usr/local/bin/grave-worker` (sh): legit maintenance loop writing `/var/lib/grave/ok` every 10s;
     boot script respawns it; also starts `cron` and seeds 5 initial zombies. Boot-ok + sleep infinity.
3. Checks per above; `necromancer-stopped` manifest timeout 90 (max allowed is 60 — NO: schema caps
   timeout_sec at 60 (SV014). Redesign: check verifies cron entry absent AND zombie count zero
   with retries within 55s; the "no respawn" guarantee comes from cron's minute-granularity —
   document in check comment that count==0 sustained 55s + entry absent is sufficient).
   MSGs guide: `The circle still glows in /etc/cron.d/…` / `Shambling things remain — count them with pgrep.`
4. Hints: 01 `ps -ef --forest`, who keeps making these? things that run "every minute" live where?
   02 `/etc/cron.d/necromancer` — remove/comment it, then clear survivors with `pkill -f zombie-worker`.
   03 near-answer incl. warning: don't pkill the gravekeeper (`grave-worker` ≠ `zombie-worker`).
5. `solution.md` (+ Lesson: kill the source first, then the symptoms; cron spawn paths; careful
   pkill patterns). `solution.sh`: `rm -f /etc/cron.d/necromancer; pkill -f zombie-worker || true;`
   verify loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: first two locks closed, gravekeeper OPEN.
- [ ] Overeager `pkill -f worker` kills gravekeeper → third lock closes with its MSG (transcript proves the trap).
- [ ] Zombie population stays < 70 and room remains responsive under pids=256 (manual observation noted).
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
