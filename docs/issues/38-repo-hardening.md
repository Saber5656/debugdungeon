# Title

Repository hardening: Dependabot, CodeQL, secret scanning, settings

## Summary

Apply the standing supply-chain and repo-security controls: dependency update automation, static
code scanning, secret scanning + push protection, and verification of branch protections.

## Context

DESIGN §10.7 (A2 supply chain). The repo is public from day one; these controls must predate any
release tag. Some switches are repo *settings* (admin) rather than files — the issue provides
exact commands and marks which need the maintainer.

## Scope

- `.github/dependabot.yml`, `.github/workflows/codeql.yml`
- Repo settings via documented `gh api` commands (executed by maintainer or with admin token)
- Not: org-level policies, rulesets redesign (verify-only)

## Detailed Requirements

1. `dependabot.yml`: weekly `gomod` (grouped minor+patch into one PR, majors separate) and weekly
   `github-actions` updates; labels `deps`; open-pr limit 5.
2. `codeql.yml`: language `go`; triggers: PR to main + weekly cron; SHA-pinned `github/codeql-action`
   steps; `permissions: { contents: read, security-events: write }` job-scoped; default query suite.
3. Secret protection (admin, via `gh api -X PATCH /repos/Saber5656/debugdungeon` payloads —
   document exact JSON): enable secret scanning, push protection, private vulnerability reporting,
   and dependency graph/Dependabot alerts. Each with a verify command (`gh api … --jq`).
4. Branch protection: verify the existing main ruleset requires PRs and blocks force-push
   (owner set this up previously); document current state via
   `gh api /repos/:owner/:repo/rulesets` output pasted into the PR; require CI checks
   (`lint`, `test`) as required status checks — add if absent (admin step).
5. Actions policy (settings): default workflow permissions = read-only; "Allow GitHub Actions to
   create and approve pull requests" = off. Verify/set via
   `gh api /repos/:owner/:repo/actions/permissions/workflow`.
6. Record final state: `docs/security/repo-settings.md` — table of control → state → verify command
   → date; audit (39) re-runs the verify column.

## Acceptance Criteria

- [ ] Dependabot opens (or would open — config validated by GitHub UI "recent updates" page) grouped PRs; config lint passes.
- [ ] CodeQL run green on main; alerts page active.
- [ ] Secret scanning + push protection + private vuln reporting: enabled, evidence via API outputs.
- [ ] Required status checks include lint+test; force-push blocked (API evidence).
- [ ] `docs/security/repo-settings.md` complete with verify commands that pass.

## Validation

All `--jq` verify commands return expected values (paste outputs in PR); one intentionally-planted
dummy secret push is BLOCKED by push protection (evidence, then removed — use a GitHub test token
pattern like `ghp_` + filler that triggers detection without being real).

## Dependencies

02.

## Non-goals

OpenSSF Scorecard badge (optional follow-up), signed commits enforcement, CODEOWNERS (solo-maintainer; revisit with contributors).

## Design References

DESIGN §10.7; SECURITY.md (03); global git rules (no direct main pushes — already enforced).
