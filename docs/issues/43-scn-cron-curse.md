# Title

Scenario: cron-curse (Floor 2 — cron environment pitfalls)

## Summary

Floor-2 room: a nightly offering job "runs" but produces nothing — its crontab line has a
percent-sign pitfall and relies on PATH/env that cron doesn't provide; the player must make the
job actually produce its artifact.

## Context

Teaches: cron's minimal environment, `%` semantics in crontab, absolute paths, redirecting job
output to see errors. DESIGN §14 Floor 2 row 4.

## Scope

`scenarios/cron-curse/` only.

## Detailed Requirements

1. `scenario.yaml` (C3): id `cron-curse`, floor 2, difficulty 2, topics `[cron, shell, env]`,
   time_estimate_min 25. Dockerfile (C3 preamble) installs `cron`. Locks:
   - `offering-made` / `A fresh offering lies on the altar` — `timeout_sec: 60`; exact algorithm
     (budget ≤ 55s): retry every 5s for any `/var/altar/offering-*.txt` whose age
     (`now − mtime`) ≤ 70s. Timing contract (check header comment): the fixed job fires every
     minute, and `solution.sh` itself waits for the first artifact before exiting, so by
     lock-time a fresh offering exists and this lock is a fast re-verify; pristine state has no
     files at all → CLOSED immediately after the retry budget.
   - `curse-lifted` / `The incantation is sound` — exact predicate over `/etc/cron.d/offering`
     non-comment lines: no `%` that is not preceded by `\` (`grep -E` for `(^|[^\\])%`), AND the
     job line references `/opt/rituals/make-offering` (absolute), AND `RITUAL_HOME=` is assigned
     (env header line or inline). Timeout 10.
2. Dockerfile:
   - `/opt/rituals/make-offering` (sh, NOT in default cron PATH): writes
     `/var/altar/offering-$(date +%s).txt` with a blessing line; requires `RITUAL_HOME` env var set
     (exits 2 with message to stderr if unset) — the env lesson.
   - Broken `/etc/cron.d/offering`:
     `*/1 * * * * root make-offering > /var/altar/log-%date.txt 2>&1` — three faults: bare command
     name (PATH), unescaped `%` (cron treats as newline/stdin → job breaks), and no `RITUAL_HOME`.
   - Cron-file validity requirements (Debian cron ignores bad files silently — these prevent
     accidental flake): `/etc/cron.d/offering` owner root:root, mode 0644, name matches cron.d
     rules (it does), trailing newline REQUIRED; the broken file must still be syntactically
     loadable (the `%` fault breaks the COMMAND, not the file). Boot: start `cron`, then C3
     boot-ok marker, `exec sleep infinity`. `/var/altar/` exists, empty.
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
