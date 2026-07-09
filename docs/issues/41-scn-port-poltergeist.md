# Title

Scenario: port-poltergeist (Floor 2 — port conflict)

## Summary

Floor-2 room: a rogue leftover process squats on port 8080, so the real warden service can't bind;
the player must identify the squatter with socket tools and evict it properly.

## Context

Teaches: `ss -tlnp`/`netstat`, `EADDRINUSE` in logs, finding a process by port, kill vs disable
(the rogue respawns if only killed — its spawner must be disabled). DESIGN §14 Floor 2 row 2.

## Scope

`scenarios/port-poltergeist/` only.

## Detailed Requirements

1. `scenario.yaml`: id `port-poltergeist`, floor 2, difficulty 2, topics `[services, network, processes]`,
   time_estimate_min 20. Install `iproute2` (for `ss`) and `curl` in the image (COOKBOOK apt rules).
   Locks:
   - `warden-listening` / `The warden holds the gate port` — `curl -fsS localhost:8080/whoami`
     returns `wardd` (retries ≤ deadline; timeout 40).
   - `poltergeist-gone` / `The poltergeist is banished` — no process named `ghostd` running AND
     its spawner disabled (check both: `pgrep ghostd` empty AND `/etc/dungeon-spawn.d/ghostd` absent or marked disabled).
2. Dockerfile:
   - `/usr/local/bin/ghostd` (sh): tiny HTTP-ish listener on 8080 answering `ghostd` to `/whoami`
     (use `busybox httpd`-free approach: a `while true; do { printf 'HTTP/1.0 200 OK\r\n\r\nghostd'; } | nc -l -p 8080 -q 1; done` loop — requires `netcat-openbsd`; add to apt list).
   - `/usr/local/bin/wardd` (sh): same shape answering `wardd`; logs bind failure
     `wardd: bind 0.0.0.0:8080: address already in use` to `/var/log/wardd.log` and retries every 5s.
   - Spawner framework: boot script iterates `/etc/dungeon-spawn.d/*` executable entries in lexical
     order, respawn-looping each; entries: `10-ghostd` (the fault: legacy entry never removed),
     `20-wardd`. Killing ghostd without removing/disabling `10-ghostd` → it returns (poltergeist!).
   - Boot-ok + sleep infinity.
3. Checks: `warden.sh` curl-based with retry; MSG points at `/var/log/wardd.log`. `ghost.sh` as
   specified; MSG when respawning: `You strike it down, yet it returns — find what summons it.`
4. Hints: 01 who owns :8080? `ss -tlnp` / read wardd's log. 02 kill it — it comes back; ghosts
   have summoners: look at how things get started (boot spawn dir). 03 remove/disable
   `/etc/dungeon-spawn.d/10-ghostd`, kill the ghost once more, wait for wardd's retry.
5. `solution.md` + Lesson Learned (ports are exclusive; fix the source, not the symptom; respawn
   supervision). `solution.sh`: `rm -f /etc/dungeon-spawn.d/10-ghostd; pkill -f ghostd || true;`
   wait-loop until curl answers `wardd` (≤ 40s).

## Acceptance Criteria

- [ ] Harness green both arches; pristine: both locks closed.
- [ ] Kill-only route demonstrably fails `poltergeist-gone` after respawn (transcript with the MSG).
- [ ] `nc`/`ss` present on both arches via apt (no arch-specific bits); image ≤ 200 MB.
- [ ] `ValidateDir` clean; no flake over 3 harness runs.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=port-poltergeist`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Real HTTP servers (48 covers nginx), firewall interference (infeasible — COOKBOOK §5).

## Design References

DESIGN §14 Floor 2; COOKBOOK §3–§6.
