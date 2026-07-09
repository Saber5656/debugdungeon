# Title

Scenario: log-labyrinth (Floor 3 — root-cause hunt in noisy logs)

## Summary

Floor-3 room: the lantern service fails at boot with a generic error, while the true cause (a bad
locale value in a config include) hides deep in rotated, noisy logs; the player must find it,
fix it, and *write down* the culprit file to prove understanding.

## Context

Teaches: log triage under noise (`grep -r`, `zgrep`, timestamps correlation), rotated-log
archaeology, include-chain configs, and articulating a root cause. The "write the answer" lock
pattern (grader file) debuts here. DESIGN §14 Floor 3 row 3.

## Scope

`scenarios/log-labyrinth/` only.

## Detailed Requirements

1. `scenario.yaml`: id `log-labyrinth`, floor 3, difficulty 3, topics `[logs, config, triage]`,
   time_estimate_min 30. Locks:
   - `lantern-lit` / `The lantern burns` — lantern process alive + status file OK (retries).
   - `cause-named` / `You named the true culprit` — `/root/answer` contains the token
     `lc_moon.conf` (grep -qi; MSG: `Write the culprit file's name into /root/answer.`).
2. Dockerfile + boot:
   - `lanternd` (sh): reads `/etc/lantern/lantern.conf` which `include /etc/lantern/conf.d/*.conf`;
     `conf.d/` holds 8 innocuous includes + `lc_moon.conf` containing `glow_locale=moon_UTF-9`
     (invalid); lanternd exits with GENERIC error `lanternd: fatal: initialization failed (code 7)`
     to stderr/log — but 40 boots ago it logged the real line
     `lanternd: locale 'moon_UTF-9' unknown (from conf.d/lc_moon.conf)` … which now lives only in
     `/var/log/lantern/lantern.log.3.gz` (pre-rotated at build: generate ~6 rotated logs, 2–4 MB
     total, filled with plausible noise incl. red herrings: a WARN about disk, a transient DNS-ish
     line, an old fixed error).
   - Current log contains only generic failures (respawn loop). `logrotate` config present as scenery.
   - Boot: respawn lanternd, boot-ok, sleep infinity. Install `gzip` (zgrep).
3. Fix: correct value `glow_locale=moon_UTF-8` (or delete the include line — both make lanternd start).
4. Hints: 01 the current log repeats a useless line — errors had ancestors: check rotated files
   (`ls /var/log/lantern/`, `zgrep`). 02 search across all rotations for `lantern` + `fatal|unknown|invalid`;
   correlate the *first* failure time. 03 `zgrep -h unknown /var/log/lantern/*.gz` → lc_moon.conf;
   fix the value; `echo lc_moon.conf > /root/answer`.
5. `solution.md` (+ Lesson: generic errors are trailheads, not causes; rotated logs; grep
   strategy; writing down root causes). `solution.sh`: sed-fix the include, write `/root/answer`, wait for lantern OK.
6. Red-herring discipline: each herring must be plausibly dismissible with in-room evidence
   (document each in solution.md's "dead ends" section — REQUIRED).

## Acceptance Criteria

- [ ] Harness green both arches; pristine: both locks closed.
- [ ] `zgrep` path proven necessary: token absent from uncompressed logs (grep -r on *.log misses it) — verified.
- [ ] Deleting the include (alt-solution) also opens `lantern-lit`; `cause-named` still requires the filename — both routes in transcript.
- [ ] solution.md contains the "dead ends" section dismissing every herring.
- [ ] `ValidateDir` clean; image ≤ 160 MB; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=log-labyrinth`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

journald (absent by design), live log streaming lessons.

## Design References

DESIGN §14 Floor 3; COOKBOOK §4, §6, §8 (herring/hint discipline).
