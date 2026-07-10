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
   - `lantern-lit` / `The lantern burns` — `/run/lantern/status` exists, contains exactly `OK`,
     and mtime < 15s (lanternd rewrites it every 5s when healthy); retry ≤ 40s, timeout 50.
     MSG: `The lantern gutters — its own words are in /var/log/lantern/.`
   - `cause-named` / `You named the true culprit` — `/root/answer` contains the token
     `lc_moon.conf` (grep -qi; MSG: `Write the culprit file's name into /root/answer.`).
2. Dockerfile + boot:
   - `lanternd` (sh; exact behavior): reads `/etc/lantern/lantern.conf`, then sources every
     `/etc/lantern/conf.d/*.conf` in glob order (KEY=VALUE lines). Validation: `glow_locale` must
     be one of `moon_UTF-8|sun_UTF-8`; any other value → log the GENERIC line
     `lanternd: fatal: initialization failed (code 7)` to `/var/log/lantern/lantern.log`, exit 1.
     (The specific `unknown locale` wording appears ONLY in the historical rotated log below —
     current builds of lanternd log generically, which IS the puzzle.) Healthy: write `OK` to
     `/run/lantern/status` every 5s.
   - `conf.d/` holds 8 innocuous includes + `lc_moon.conf` containing `glow_locale=moon_UTF-9`.
   - Rotated-log fabrication (build-time generator script, deterministic): exactly
     `lantern.log.1` (plain) and `lantern.log.{2..5}.gz`; each 200–400 KB of templated noise lines
     with fixed pseudo-timestamps; the true clue line
     `lanternd: locale 'moon_UTF-9' unknown (from conf.d/lc_moon.conf)` sits in `lantern.log.3.gz`
     at a generator-fixed position; red herrings (one disk WARN in .2.gz, one transient
     resolver error in .4.gz, one already-fixed permission error in .5.gz).
   - Current log: generic failures only, appended by the respawn loop; respawner sleeps 5s between
     attempts and truncates the current log at 1 MB (bounded growth).
   - Boot (C3): start the respawn loop, boot-ok marker, sleep infinity. Install `gzip` (zgrep).
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
- [ ] Rotated-log route verified: the explicit clue line exists ONLY in `lantern.log.3.gz` (absent
  from all plain `*.log` — `grep -r` on uncompressed files misses it; `zgrep` finds it). Config-dir
  inspection remains a legitimate alternate diagnosis route (the culprit value is, necessarily,
  in `lc_moon.conf` itself) — the `cause-named` lock accepts either route since both name the file.
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
