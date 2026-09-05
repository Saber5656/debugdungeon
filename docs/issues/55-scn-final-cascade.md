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
   - `waters-receded` / `The flood has receded` — tmpfs usage ≤ 70% AND flooder disabled
     (`/etc/chronicler.enabled` absent AND no `chronicler` process running — same both-conditions
     rule as issue 44, so it can't be satisfied transiently).
   - `herald-stands` / `The herald stands again` — heraldd running >10s, no stale pidfile conflict.
   - `throne-answers` / `The throne answers petitions` — end-to-end probe using the exact
     petition contract below: submit via `throne-petition "audit probe"`, then retry ≤ 45s
     (timeout 50) for `/var/spool/throne/answered/<petition-id>.txt` containing an `ANSWERED:`
     line.
   - `chronicle-written` / `You chronicled the cascade` — `/root/chronicle` names all three causes:
     must match three tokens: `flood|debug.log`, `pidfile`, `permissions|chmod|chown` (grep triple; MSG lists missing count, not which).
2. Petition contract (exact, self-contained): `/usr/local/bin/throne-petition "<text>"` writes
   `/var/spool/throne/inbox/p-<epoch>-<rand4>.txt` (0644, content = the text), prints the petition
   id (`p-…`) to stdout. `heraldd` (user `herald`) polls inbox every 5s; for each petition it
   writes `/var/spool/throne/answered/<same-name>` containing `ANSWERED: <text>` then removes the
   inbox file.
3. Cascade construction (Dockerfile + boot; boot owns ALL tmpfs content — 44's pattern; each fault
   gated behind the previous):
   - **Fault A (flood)**: boot seeds `/var/spool/throne/debug.log` to ENOSPC (44's dd pattern);
     `chronicler` loop (`/usr/local/bin/chronicler`) runs FOREVER until killed — the flag controls
     WRITES only, not process lifetime (44's corrected pattern:
     `while true; do [ -e /etc/chronicler.enabled ] && { append || true; }; sleep 2; done`), so
     `waters-receded`'s process clause is stable.
   - **Fault B (pidfile)**: `heraldd` startup guard — if `/run/herald.pid` exists → log
     `heraldd: refusing to start: pidfile /run/herald.pid exists (stale?)` and exit 1; ALSO
     requires ≥ 20% free space on the spool (`df -P` check) → logs
     `heraldd: spool has no room to work` (so A must be fixed first). `/run/herald.pid` baked
     stale at boot (contains pid 99999). Respawn loop: 5s interval, log `/var/log/heraldd.log`.
   - **Fault C (perms)**: boot creates `/var/spool/throne/{inbox,answered}` — inbox 1777,
     `answered` root:root 0700 while heraldd runs as `herald` → processing fails at the LAST step
     with `heraldd: cannot write answered/: permission denied` (only observable once A+B fixed —
     true cascade). Fix: `chown herald:herald /var/spool/throne/answered && chmod 0750 …`.
   - Boot order: spool dirs → flood seed → stale pidfile → chronicler + heraldd respawn loops →
     C3 boot-ok → sleep infinity. Boot < 25s.
4. Fix order: A (remove flag + pkill chronicler + truncate debug.log) → B (remove stale pidfile;
   heraldd respawns within 5s) → C (chown/chmod answered/) → probe passes → write chronicle.
   Wrong-order probes and their expected evidence (transcripts): B-before-A → heraldd log shows
   the no-room line; C invisible before B → heraldd.log has no permission-denied line yet.
5. Hints (4, wider spacing for difficulty 5): 01 start from the symptom (`throne-petition` fails)
   and walk DOWN the stack: app → service → disk; every layer leaves words in /var/log.
   02 heraldd names two blockers — one is space, one is a leftover from a crash; earlier floors
   taught both. 03 after heraldd runs: watch its log at the LAST step — who may write where?
   04 near-answer sequence of the three fixes + chronicle format.
6. `solution.md` (+ Lesson: cascades unwind bottom-up, one-cause-at-a-time verification, chronicle
   discipline; dead-ends section per 46's pattern). `solution.sh` (exact sequence):
   ```sh
   rm -f /etc/chronicler.enabled; pkill -f /usr/local/bin/chronicler || true
   : > /var/spool/throne/debug.log
   rm -f /run/herald.pid
   chown herald:herald /var/spool/throne/answered; chmod 0750 /var/spool/throne/answered
   printf 'flood in debug.log\nstale pidfile\nanswered dir permissions\n' > /root/chronicle
   id=$(throne-petition "solution probe"); # retry ≤ 60s for answered/$id.txt
   ```

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

12, 29. (Mechanics from sibling rooms 32/40/44/46 are INLINED above — self-contained per
ISSUE_PLAN's mutual-independence rule; the sibling references are pedagogical only.)

## Non-goals

Database involvement (kept to floors), time pressure mechanics, multiple endings.

## Design References

DESIGN §14 Capstone, §3.5 (unlock ≥12); COOKBOOK §3–§6, §13.
