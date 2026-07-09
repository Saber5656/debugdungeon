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

1. `harness.TestScenario(ctx, api, l *scenario.Loaded) (Report, error)` steps (each timed):
   1. `EnsureImage` (14).
   2. `CreateRoom` + `InjectHelpers` + `StartRoom` (15/18) with a `harness-<id>` runID; wait for
      boot: poll `/var/dungeon/boot-ok` via exec (≤ 30s; cookbook §3 requires the marker) — missing
      after 30s → FAIL `boot-timeout`.
   3. `locks.RunAll` → require **≥ 1 lock CLOSED**; all-open → FAIL `already-open` (breakage absent).
   4. Pipe `solution.sh` (root, `/bin/sh -s`, 300s budget, output captured to report).
   5. `locks.RunAll` → require **ALL OPEN**; else FAIL `unsolved` listing closed locks + their MSG.
   6. Teardown always (remove container; keep image for cache).
   - `Report{ScenarioID, Arch, ImageSizeBytes, Steps []StepResult, Pass bool, FailReason string}`.
2. Size check: after build, `ImageInspect` size vs budget (500 MB; 700 MB for floor 5 — read
   budget from a map in the harness keyed by floor) → over-budget = FAIL `oversize`.
3. `scenarios_solvability_test.go`: iterates `Registry.All()` (bundled), subtests per scenario,
   honoring `-run` filtering; env `DD_SCENARIO_FILTER` (comma ids) for CI path-filtering.
4. `.github/workflows/scenarios.yml` (hardening rules identical to issue 02):
   - Triggers: `pull_request` touching `scenarios/**` or engine paths
     (`internal/{dockerx,locks,scenario,harness}/**`), plus weekly cron (full sweep), plus manual dispatch.
   - Matrix: `runs-on: [ubuntu-latest, ubuntu-24.04-arm]` (KU-1: if the arm runner is unavailable
     to this repo at implementation time, drop it from PR triggers, keep it in cron, and note the
     deviation in this issue's PR).
   - Steps: checkout → setup-go → compute changed scenario ids (`git diff --name-only` vs base →
     unique top-level dirs under `scenarios/`) → export `DD_SCENARIO_FILTER` (PR) or empty (cron)
     → `make itest-scenarios` (`go test -tags itest -run Solvability -timeout 45m ./...`).
   - Disk hygiene: `docker system prune -af` between scenarios when free disk < 4 GB (harness
     checks and prunes managed images oldest-first; simpler: harness removes the image after test
     when env `DD_HARNESS_PRUNE=1`, set in CI).
5. Failure output must be actionable: print the failing step, closed lock ids, and last 30 lines
   of solution output.

## Acceptance Criteria

- [ ] Harness FAILs correctly on three sabotaged fixtures: no boot marker, no initial breakage, broken solution (fixtures under `internal/harness/testdata`).
- [ ] Harness PASSes `scenarios/_template`.
- [ ] Workflow runs on a PR touching a fixture scenario and skips on engine-unrelated docs PRs; cron entry present.
- [ ] Matrix includes arm64 (or documented KU-1 fallback applied).
- [ ] All actions SHA-pinned; `permissions: contents: read`.

## Validation

`make itest` incl. harness fixture tests; a scratch PR demonstrating the workflow triggering and
passing on `_template`.

## Dependencies

02, 10, 14, 15, 19 (18 for boot/injection).

## Non-goals

User-facing command UX (57), performance benchmarking, flaky-retry logic (a flaky scenario is a content bug).

## Design References

DESIGN §6.4, §12; ADR-006; KU-1, KU-5.
