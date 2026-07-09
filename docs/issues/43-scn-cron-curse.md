# Title

Scenario: cron-curse (Floor 2 — cron environment pitfalls)

## Summary

Floor-2 room: a nightly offering job "runs" but produces nothing — its crontab line has a
percent-sign pitfall and relies on PATH/env that cron doesn't provide; the player must make the
job actually produce its artifact.

## Context

Teaches: cron's minimal environment, `%` semantics in crontab, absolute paths, redirecting job
output to see errors, `run-parts` vs crontab. DESIGN §14 Floor 2 row 4.

## Scope

`scenarios/cron-curse/` only.

## Detailed Requirements

1. `scenario.yaml`: id `cron-curse`, floor 2, difficulty 2, topics `[cron, shell, env]`,
   time_estimate_min 25. Install `cron`. Locks:
   - `offering-made` / `A fresh offering lies on the altar` — `/var/altar/offering-*.txt` exists
     with mtime < 120s… (SV014 caps at 60; instead: lock checks a *state file* the job writes:
     newest offering mtime < 90s is not checkable in 60s… REDESIGN for testability: the fixed
     job runs every minute (`* * * * *`), so: retry up to 55s for ANY offering file newer than
     70s ago — max staleness after fix is 60s + write time, so a 55s retry window starting
     post-solution suffices; solution.sh itself waits for the first artifact before exiting,
     making the lock a fast re-verify. Document this timing contract in the check header.)
   - `curse-lifted` / `The incantation is sound` — crontab line no longer contains an unescaped `%`
     AND references the script by absolute path (grep-based on `/etc/cron.d/offering`).
2. Dockerfile:
   - `/opt/rituals/make-offering` (sh, NOT in default cron PATH): writes
     `/var/altar/offering-$(date +%s).txt` with a blessing line; requires `RITUAL_HOME` env var set
     (exits 2 with message to stderr if unset) — the env lesson.
   - Broken `/etc/cron.d/offering`:
     `*/1 * * * * root make-offering > /var/altar/log-%date.txt 2>&1` — three faults: bare command
     name (PATH), unescaped `%` (cron treats as newline/stdin → job breaks), and no `RITUAL_HOME`.
   - `/var/altar/` exists, empty; cron started by boot script; boot-ok; sleep infinity.
   - `/var/log/syslog`-less: install `rsyslog`? NO — keep light: cron's own mail is absent; the
     teaching moment is redirecting output yourself. Provide `/var/log/cron-hint.log` breadcrumb
     via a boot-time note in `/root/tavern-rumors.txt` ("the abbot swears the ritual is scheduled…").
3. Checks: `offering.sh` per timing contract above (MSG: `The altar stays bare — does the ritual
   even run? Make its errors visible.`); `incantation.sh` greps the cron file for `%` (unescaped)
   and non-absolute command (MSG accordingly).
4. Hints: 01 cron jobs get a *tiny* environment: no your-PATH, no your-vars; also `%` is special
   in crontabs. 02 rewrite the line: absolute path, `\%` or avoid `%`, set `RITUAL_HOME=/opt/rituals`
   (env line in cron.d or inline), redirect output to a log you can read. 03 near-answer full line:
   `RITUAL_HOME=/opt/rituals` header + `* * * * * root /opt/rituals/make-offering >> /var/altar/ritual.log 2>&1`.
5. `solution.md` (+ Lesson: cron env/`%`/absolute paths/observability-first debugging).
   `solution.sh`: rewrite `/etc/cron.d/offering` (heredoc), then wait ≤ 75s for the first offering file.

## Acceptance Criteria

- [ ] Harness green both arches (solution.sh's internal wait absorbs cron's minute tick; harness 300s budget fine).
- [ ] Pristine: both locks closed; after fixing only `%` (not PATH/env) offering still absent — layered discovery shown in transcript.
- [ ] All lock timeouts ≤ 60; no flake ×3 (cron timing tolerated by the documented contract).
- [ ] `ValidateDir` clean; image ≤ 150 MB.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=cron-curse`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

anacron/systemd-timers, mail spool lessons.

## Design References

DESIGN §14 Floor 2; COOKBOOK §4, §6 (timing contract pattern); issue 08 SV014.
