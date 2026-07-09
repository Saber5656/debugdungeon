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
   - `wraith-risen` / `The wraith stirs` — `redis-cli -h 127.0.0.1 ping` → `PONG` (retries).
   - `memory-restored` / `The wraith remembers` — `redis-cli get soul:0001` → `intact` (the
     pre-corruption sentinel key survives repair).
   - `keepsake-kept` / `Persistence still guards the memory` — `redis-cli config get appendonly` → `yes`
     (guards the "just disable AOF" shortcut).
2. Dockerfile + boot:
   - Build-time: start redis briefly with `appendonly yes`, SET `soul:0001 intact` + ~100 filler
     keys, clean shutdown; then **truncate the AOF mid-command**: `truncate -s -17` the appendonly
     file (bookworm redis 7: files under `appendonlydir/` — truncate the last `.incr.aof`; verify
     exact layout at implementation, document).
   - Config: `appendonly yes`, `dir /var/lib/redis`, bind 127.0.0.1, `aof-load-truncated no`
     (IMPORTANT: default `yes` would silently self-heal — setting `no` forces the refusal and the lesson; call this out).
   - Boot: respawn-loop redis-server (fails, logging `Bad file format reading the append only file`
     to `/var/log/redis/redis.log`), boot-ok, sleep infinity.
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
- [ ] Truncation point verified to corrupt (not merely shorten past a command boundary) — build asserts redis refuses to start pristine.
- [ ] `ValidateDir` clean; image ≤ 250 MB; no flake ×3; redis-7 appendonlydir layout documented.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=redis-wraith`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

RDB snapshots, cluster/sentinel, memory-pressure tuning.

## Design References

DESIGN §14 Floor 5; COOKBOOK §3–§4.
