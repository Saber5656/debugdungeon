# Title

Scenario: redis-wraith (Floor 5 — corrupted AOF repair)

## Summary

Floor-5 room: the wraith's memory (Redis with AOF persistence) refuses to rise after a truncated
append-only file; the player must diagnose the refusal, repair the AOF with the bundled tooling,
and verify the precious keys survived.

## Context

Teaches: reading redis startup refusals, `redis-check-aof --fix`, persistence trade-offs, data
verification after repair. DESIGN §14 Floor 5 row 2; difficulty 4.

## Scope

`scenarios/redis-wraith/` only.

## Detailed Requirements

1. `scenario.yaml`: id `redis-wraith`, floor 5, difficulty 4, topics `[redis, database, persistence]`,
   time_estimate_min 30. Install `redis-server`, `redis-tools`. Locks:
   - `wraith-risen` / `The wraith stirs` — `redis-cli -h 127.0.0.1 ping` → `PONG` (retries; a
     living server also proves the AOF now loads clean, since `aof-load-truncated no` means a still-
     corrupt AOF keeps redis DOWN → this lock CLOSED).
   - `memory-restored` / `The wraith remembers` — `redis-cli get soul:0001` → `intact` (the
     pre-corruption sentinel key survives repair).
   - `keepsake-kept` / `Persistence still guards the memory` — FILE-based (works while redis is
     down — `redis-cli config get` can't run against a dead server): `/etc/redis/redis.conf`
     non-comment lines contain `appendonly yes` AND `aof-load-truncated no` AND no `appendonly no`
     AND no `aof-load-truncated yes`. This closes the DESIGN §14 "AOF loads clean" intent AND the
     re-review's escape (flipping `aof-load-truncated yes` would let redis silently skip the
     corrupt tail — forbidden here; the player must actually repair the AOF, not disable the
     safety). Pristine (config untouched: `yes`/`no`) → OPEN. Timeout 10.
2. Dockerfile + boot (C3 preamble; apt: `redis-server`, `redis-tools`):
   - Config `/etc/redis/redis.conf` (exact keys): `appendonly yes`, `dir /var/lib/redis`,
     `appenddirname "appendonlydir"`, `bind 127.0.0.1`, `daemonize no`,
     `logfile /var/log/redis/redis.log`, `aof-load-truncated no` (IMPORTANT: default `yes` would
     silently self-heal — `no` forces the refusal and the lesson; call this out in solution.md).
   - Build-time: start `redis-server /etc/redis/redis.conf &` briefly AS the redis user (dirs
     chowned redis:redis), SET `soul:0001 intact` + ~100 filler keys, clean `SHUTDOWN SAVE`; then
     **corrupt the AOF**: redis-7 layout is `/var/lib/redis/appendonlydir/appendonly.aof.1.incr.aof`
     (+ base/manifest) — `truncate -s -17` the `.incr.aof`, and the build MUST then assert the
     corruption bites: `redis-server /etc/redis/redis.conf` exits non-zero with `Bad file format`
     within 10s (build fails otherwise — guarantees pristine-closed locks; review catch).
   - Boot (C3): respawn loop `while true; do su -s /bin/sh redis -c 'redis-server /etc/redis/redis.conf' >>/var/log/redis/boot.log 2>&1; sleep 5; done &`,
     C3 boot-ok marker, sleep infinity.
3. Fix: read log → `redis-check-aof --fix <file>` (answer `y`; note the tool warns about data-loss
   tail) → redis starts via respawn → verify keys.
4. Hints: 01 the wraith dies at birth — its log names the exact ailment; which file does it choke
   on? 02 redis ships a healer for that file: look at `redis-check-aof`; it must run against the
   file redis actually loads (find `dir` + appendonlydir). 03 near-answer: the exact
   `redis-check-aof --fix` invocation, then wait for the respawn loop and `redis-cli ping`.
5. `solution.md` (+ Lesson: AOF anatomy, why aof-load-truncated matters, repair-tool tail-loss
   trade-off, verify-after-repair discipline; dead end: deleting the AOF also revives the wraith
   but loses ALL memory → `memory-restored` stays closed — the point). `solution.sh`: scripted
   `redis-check-aof --fix` (pipe `y`), wait, verify sentinel key.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: locks 1–2 closed, lock 3 OPEN (appendonly still yes).
- [ ] Delete-the-AOF route → wraith rises but `memory-restored` stays closed (transcript).
- [ ] `appendonly no` route → `keepsake-kept` closes (transcript).
- [ ] Truncation verified to corrupt at BUILD time (build fails if redis would start clean) — the assert step above.
- [ ] `ValidateDir` clean; image ≤ 250 MB; no flake ×3; exact AOF path (`appendonlydir/appendonly.aof.1.incr.aof`) confirmed against the shipped redis version and recorded in solution.md.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=redis-wraith`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

RDB snapshots, cluster/sentinel, memory-pressure tuning.

## Design References

DESIGN §14 Floor 5; COOKBOOK §3–§4.
