# Title

Scenario solvability test harness and CI matrix

## Summary

Implement the ADR-006 gate: an internal harness that proves, for every bundled scenario, that the
breakage exists and `solution.sh` opens all locks under the production security profile — wired
into a CI matrix on amd64 and arm64.

## Context

This gate is what makes 16 content issues (30–33, 40–55) mechanically verifiable. It reuses the
production engine pieces (14, 15, 19) — no parallel implementation.

## Scope

- `internal/harness/harness.go` + `scenarios_solvability_test.go` (build tag `itest`)
- `.github/workflows/scenarios.yml`
- Not: the user-facing `scenario test` command (57 wraps this)

## Detailed Requirements

1. `harness.TestScenario(ctx, deps Deps, l *scenario.Loaded, o Options) (Report, error)` where
   `Deps{API dockerx.API; Reg *scenario.Registry; Out io.Writer}` (Out gets sanitized progress
   lines; `textsafe.Sanitize` on ALL container/build-derived bytes before Out/report — TB2/TB3)
   and `Options{KeepOnFailure bool; SolutionTimeout time.Duration /* default 300s, max 900s */;
   PruneImageAfter bool}`; `Report` additionally carries `ContainerID, ImageRef string` (consumed
   by 57's `--keep`). Steps (each timed):
   1. `EnsureImage` (14).
   2. `CreateRoom` + `InjectHelpers` + `StartRoom` (15/18) with a `harness-<id>` runID; wait for
      boot: poll `/var/dungeon/boot-ok` via exec (≤ 30s; cookbook §3 requires the marker) — missing
      after 30s → FAIL `boot-timeout`.
   3. `locks.RunAll` → require **≥ 1 lock CLOSED**; all-open → FAIL `already-open` (breakage absent).
   4. Run `solution.sh` with the LOCK-RUNNER exec contract (19): root, non-TTY,
      `timeout <SolutionTimeout> /bin/sh -s` stdin-piped, output captured (≤ 64 KiB, sanitized)
      to the report; exit ≠ 0 → FAIL `solution-exit-nonzero`; timeout → FAIL `solution-timeout`.
   5. `locks.RunAll` → require **ALL OPEN**; else FAIL `unsolved` listing closed locks + their MSG.
   6. Teardown: remove container (skipped when `KeepOnFailure` and the run FAILed — ids in the
      report); remove image when `PruneImageAfter` (CI disk hygiene), else keep for cache.
   - `Report{ScenarioID, Arch string; ImageSizeBytes int64; Steps []StepResult; Pass bool;
     FailReason string; ContainerID, ImageRef string}`.
2. Size check: after build, `ImageInspect` size vs budget keyed by `l.Spec.Floor`
   (`map[int]int64`: floors 1–4 & 6 → 500 MB, floor 5 → 700 MB; fixtures/floor-1 default applies)
   → over-budget = FAIL `oversize`.
3. `scenarios_solvability_test.go`: iterates `Registry.All()` (bundled), subtests per scenario,
   honoring `-run` filtering; env `DD_SCENARIO_FILTER` (comma ids) for CI path-filtering.
4. **Service-pattern feasibility fixtures (KU-11 gate)**: under `internal/harness/testdata/feasibility/`,
   five minimal scenario dirs, each shaped identically: floor 1, ★1, ONE probe lock + a trivial
   marker breakage (boot writes `/var/dungeon/SEAL`; `solution.sh` = `rm -f /var/dungeon/SEAL` —
   satisfies the ≥1-closed gate without touching the service), so the probe lock passing on
   pristine state IS the feasibility proof:
   - `feas-su`: probe `su -l root -c true` exits 0.
   - `feas-cron`: boot starts `cron`; probe waits ≤ 55s (timeout 60) for a `* * * * *` job's artifact.
   - `feas-nginx`: apt nginx; boot starts it; probe curls 200 on 127.0.0.1:80.
   - `feas-postgres`: apt postgresql; initdb'd at build; probe `pg_isready` + trivial SELECT.
   - `feas-redis`: apt redis-server; probe `redis-cli ping` → PONG.
   Run through TestScenario in the itest suite BEFORE Wave 6 content is authored; a failing
   fixture blocks Wave 6 and triggers a design escalation (do NOT weaken the profile unilaterally —
   DESIGN §10.3).
5. `.github/workflows/scenarios.yml` (hardening rules identical to issue 02):
   - Triggers: `pull_request` touching `scenarios/**` or engine paths
     (`internal/{dockerx,locks,scenario,harness}/**`), plus weekly cron (full sweep), plus manual dispatch.
   - Matrix: `runs-on: [ubuntu-latest, ubuntu-24.04-arm]` (KU-1: if the arm runner is unavailable
     to this repo at implementation time, drop it from PR triggers, keep it in cron, and note the
     deviation in this issue's PR).
   - Steps: checkout → setup-go → decide scope: PR touching ONLY `scenarios/**` → changed ids
     (`git diff --name-only` vs base → unique top-level dirs) exported as `DD_SCENARIO_FILTER`;
     PR touching any engine path (`internal/{dockerx,locks,scenario,harness}/**`) or cron/dispatch
     → FULL sweep (empty filter) → `make itest-scenarios`
     (`go test -tags itest -run Solvability -timeout 45m ./...`).
   - Disk hygiene: CI sets `DD_HARNESS_PRUNE=1`, which the test suite maps to
     `Options.PruneImageAfter=true` (each scenario's image removed after its subtest).
6. Failure output must be actionable: print the failing step, closed lock ids, and last 30 lines
   of solution output.

## Acceptance Criteria

- [ ] Harness FAILs correctly on three sabotaged fixtures: no boot marker, no initial breakage, broken solution (fixtures under `internal/harness/testdata`).
- [ ] Harness PASSes `scenarios/_template`.
- [ ] All five feasibility fixtures PASS on amd64 and arm64 (KU-11 evidence recorded in the PR); any failure is escalated, not worked around.
- [ ] Workflow runs on a PR touching a fixture scenario and skips on engine-unrelated docs PRs; cron entry present.
- [ ] Matrix includes arm64 (or documented KU-1 fallback applied).
- [ ] All actions SHA-pinned; `permissions: contents: read`.

## Validation

`make itest` incl. harness fixture tests; a scratch PR demonstrating the workflow triggering and
passing on `_template`.

## Dependencies

02, 10, 12 (boot contract/COOKBOOK §3), 14, 15, 18, 19.

## Non-goals

User-facing command UX (57), performance benchmarking, flaky-retry logic (a flaky scenario is a content bug).

## Design References

DESIGN §6.4, §7.4, §7.6, §7.7, §12; COOKBOOK §3; ADR-006; KU-1, KU-5, KU-11.
