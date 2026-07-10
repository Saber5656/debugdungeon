# Title

Add LICENSE (MIT), CONTRIBUTING, SECURITY policy, and Code of Conduct

## Summary

Create the four community/security baseline documents for a public OSS repository.

## Context

The repo is already public. License choice is decided (MIT, owner decision 2026-07-10, DESIGN
§1.2). SECURITY.md is part of the security model (DESIGN §10.7) and referenced by the release
audit (issue 39).

## Scope

Four root files: `LICENSE`, `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`. Nothing else.

## Detailed Requirements

1. `LICENSE`: verbatim MIT text, `Copyright (c) 2026 Saber5656`.
2. `CONTRIBUTING.md` sections:
   - Dev setup: Go 1.25+ supported for contributors, Go 1.26.x is what go.mod/CI use (DESIGN §4.3);
     Docker for `make itest`; `make build/test/lint`.
   - PR rules: small PRs, one issue per PR, Conventional Commits *recommended*, DESIGN.md must be
     updated in the same PR when behavior changes (ISSUE_PLAN §6.5), no direct pushes to `main`.
   - Scenario contributions: pointer to `docs/COOKBOOK.md` (issue 12) and the solvability gate (ADR-006).
   - Issue workflow: GitHub Issues are derived from `docs/issues/`; propose spec changes against docs first.
3. `SECURITY.md`:
   - Report vulnerabilities via GitHub **Private Vulnerability Reporting** (no public issues).
     Until issue 38 confirms the setting is enabled, include the fallback line "or contact the
     maintainer via the email on the GitHub profile"; issue 38 removes the fallback.
   - Target: acknowledge ≤ 7 days, fix or public advisory ≤ 90 days.
   - Supported versions: latest minor only.
   - Scope note: scenario containers are a hardening boundary — sandbox-escape reports are in
     scope; "player can read solutions locally" is explicitly out of scope (DESIGN §2.3 anti-cheat non-goal).
4. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 verbatim from
   https://www.contributor-covenant.org/version/2/1/code_of_conduct/ ; enforcement contact text =
   "the repository owner via GitHub (https://github.com/Saber5656)".

## Acceptance Criteria

- [ ] All four files exist at repo root with the exact contents/policies above.
- [ ] `gh api repos/:owner/:repo/community/profile` shows exactly these four detected: `license`, `code_of_conduct`, `contributing`, `security_policy` (other checklist items out of scope).
- [ ] SECURITY.md scope section distinguishes in-scope (sandbox escape, pack extraction, ANSI injection) from out-of-scope (local spoilers).

## Validation

Screenshot or `gh api repos/:owner/:repo/community/profile` output showing 100% (or all four
detected) attached to the PR.

## Dependencies

None.

## Non-goals

Enabling private vulnerability reporting in repo settings (issue 38 does settings), issue templates (37/38).

## Design References

DESIGN §1.2, §2.3, §10.7; ADR-007.
