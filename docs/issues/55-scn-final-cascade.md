# Title

Scenario: final-cascade (Capstone — The Cascade Throne, multi-fault finale)

## Summary

The capstone room: three faults across layers cascade into one dead application — a full tmpfs
starves a service whose stale pidfile then blocks restart, while a permissions regression breaks
the final handoff; fixing order matters. Difficulty 5, four locks.

## Context

The graduation exam composing skills from Floors 1–5: disk triage (44), service/pidfile forensics
(40), permissions least-privilege (32), plus the write-your-answer discipline (46).
DESIGN §14 capstone; unlocks at ≥12 total clears (§3.5).

## Scope

`scenarios/final-cascade/` only.

## Detailed Requirements

1. `scenario.yaml`: id `final-cascade`, floor 6, difficulty 5, topics `[triage, disk, services, permissions]`,
   time_estimate_min 60; `mounts.tmpfs: [{path: /var/spool/throne, size_mb: 48}]`. Locks:
   - `waters-receded` / `The flood has receded` — tmpfs usage ≤ 70% AND flooder disabled.
   - `herald-stands` / `The herald stands again` — heraldd running >10s, no stale pidfile conflict.
   - `throne-answers` / `The throne answers petitions` — end-to-end probe: petition submitted via
     `throne-petition` CLI lands processed in `/var/spool/throne/answered/` (retries).
   - `chronicle-written` / `You chronicled the cascade` — `/root/chronicle` names all three causes:
     must match three tokens: `flood|debug.log`, `pidfile`, `permissions|chmod|chown` (grep triple; MSG lists missing count, not which).
2. Cascade construction (Dockerfile + boot; each fault gated behind the previous):
   - **Fault A (flood)**: `chronicler` loop floods `/var/spool/throne/debug.log` to ~46 MB
     (guarded), flag-file disable like 44.
   - **Fault B (pidfile)**: `heraldd` refuses to start while `/run/herald.pid` exists with a
     nonexistent pid (baked stale); AND it needs ≥ 20% free space on the spool to start (so A must
     be fixed first) — its log states both conditions clearly on each respawn attempt.
   - **Fault C (perms)**: `/var/spool/throne/answered/` — created by boot as root:root 0700 while
     heraldd runs as user `herald` (created at build) → processing fails at the last step with
     `permission denied` in heraldd's log (only observable once A+B fixed — true cascade).
   - Boot: create spool dirs (the tmpfs population pattern from 44), seed flood, start
     chronicler + heraldd respawn loops, boot-ok, sleep infinity. `throne-petition` CLI writes a
     petition file owned appropriately.
3. Fix order: A (disable flooder, truncate) → B (remove stale pidfile; heraldd respawns) →
   C (chown herald / fix mode 0750 herald:herald) → probe passes → write chronicle.
4. Hints (4, wider spacing for difficulty 5): 01 start from the symptom (`throne-petition` fails)
   and walk DOWN the stack: app → service → disk; every layer leaves words in /var/log.
   02 heraldd names two blockers — one is space, one is a leftover from a crash; earlier floors
   taught both. 03 after heraldd runs: watch its log at the LAST step — who may write where?
   04 near-answer sequence of the three fixes + chronicle format.
5. `solution.md` (+ Lesson: cascades unwind bottom-up, one-cause-at-a-time verification, chronicle
   discipline; dead-ends section per 46's pattern). `solution.sh`: the three fixes in order + chronicle write + petition verify loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all four locks closed.
- [ ] Fault gating verified: B unfixable before A (herald's space check), C invisible before B (no heraldd log line yet) — transcripts.
- [ ] Wrong-order attempts produce guiding MSGs, not dead ends (each lock's MSG points down-stack).
- [ ] Chronicle token matcher accepts reasonable phrasings (case-insensitive, the three token groups) — table-tested check script.
- [ ] `ValidateDir` clean; image ≤ 200 MB; boot < 25s; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=final-cascade`; full playthrough transcript (intended
path + one wrong-order probe) in PR.

## Dependencies

12, 29 (design-siblings: 32, 40, 44, 46).

## Non-goals

Database involvement (kept to floors), time pressure mechanics, multiple endings.

## Design References

DESIGN §14 Capstone, §3.5 (unlock ≥12); COOKBOOK §3–§6, §13.
