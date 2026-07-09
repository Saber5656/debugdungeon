# Title

`scenario validate` and `scenario test` commands

## Summary

Expose the validator (08) and the solvability harness (29) as user-facing commands over external
scenario directories, with human-readable reports and stable exit codes.

## Context

Authors need the exact same gates CI runs, locally, pre-submission (ADR-006; DESIGN §5.1). Both
commands must consume a plain directory (not the embedded bundle).

## Scope

- `internal/cli/scenario_validate.go`, `internal/cli/scenario_test.go`
- Registry: ensure `LoadDir(osRoot string)` external-load path exists (10 provides pack loading; reuse)
- Not: init (56), pack install (59)

## Detailed Requirements

1. `scenario validate <dir>`:
   - Load via `LoadSpec` + `ValidateDir` (strictest variant, 08).
   - Output: one line per violation `SVnnn  <path>: <msg>` (sorted, sanitized paths), summary
     `N violation(s)` — or `valid ✓ (id: <id>, floor F, ★D, locks: k, hints: m)`.
   - Exit codes: 0 valid; 4 violations or unparsable manifest; 1 I/O errors.
   - `--json` flag: machine-readable `{violations: [{rule, path, msg}]}` (authors' editor tooling; contract frozen).
2. `scenario test <dir>`:
   - Pre-flight: validate first (fail fast, same output as above, exit 4).
   - Then run `harness.TestScenario` (29) against the external dir (docker required → exit 3 via 13's preflight).
   - Report each step live: `build… ok (42s, 187 MB)` / `pristine locks… 2 closed ✓` /
     `solution.sh… ok (12s)` / `final locks… all open ✓` → `PASS`.
   - Failure prints the harness FailReason + closed locks + last 30 solution-output lines; exit 1.
   - Flags: `--keep` (don't remove container/image on failure, print ids for manual poking),
     `--timeout <s>` (solution budget override, default 300, max 900).
3. Both commands must never touch the player's progress/run state (arch test: no progress/run store imports).
4. Sanitization: all scenario-sourced strings in reports pass through 11 (validate output may name
   hostile files).

## Acceptance Criteria

- [ ] `validate` on: valid dir (0), corpus bad dirs (4, correct rule lines), missing dir (1) — table-tested.
- [ ] `--json` schema golden-tested.
- [ ] itest: `test` on `_template` copy → PASS with live step lines; sabotaged solution → FAIL + evidence lines + exit 1; `--keep` leaves the container (then cleaned).
- [ ] Docker-down `test` → exit 3 with doctor hint.
- [ ] No progress/run store imports (lint/arch test).

## Validation

`make test` + `make itest`; transcripts (pass + fail + --keep) in PR.

## Dependencies

08, 29 (10's LoadDir, 13 preflight).

## Non-goals

Batch-testing multiple dirs (shell loop suffices), watch mode, CI-format annotations (v2 nicety).

## Design References

DESIGN §5.1, §6.4, §12; ADR-006.
