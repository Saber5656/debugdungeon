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
   `github-actions` updates; labels `deps` (create first: `gh label create deps --color 6f42c1 || true`);
   open-pr limit 5.
2. `codeql.yml`: language `go`; triggers: PR to main + push to main + weekly cron +
   `workflow_dispatch` (so "green on main" is achievable immediately); ALL actions SHA-pinned with
   version comments (checkout/setup included — issue 02's rule applies to every workflow);
   `permissions: { contents: read, security-events: write }` job-scoped; default query suite.
3. Secret protection (admin; exact commands — run by maintainer or with an admin token):
   ```sh
   gh api -X PATCH repos/Saber5656/debugdungeon \
     -f security_and_analysis[secret_scanning][status]=enabled \
     -f security_and_analysis[secret_scanning_push_protection][status]=enabled
   gh api -X PUT repos/Saber5656/debugdungeon/private-vulnerability-reporting
   gh api -X PUT repos/Saber5656/debugdungeon/vulnerability-alerts        # Dependabot alerts
   # verify:
   gh api repos/Saber5656/debugdungeon --jq '.security_and_analysis'
   gh api repos/Saber5656/debugdungeon/private-vulnerability-reporting --jq '.enabled'
   ```
   (Endpoint shapes verified against current GitHub REST docs at implementation per C6; adjust if
   the API moved, recording the replacement commands in `docs/security/repo-settings.md`.)
4. Branch protection: VERIFY-ONLY for the existing main ruleset (PR-required, force-push blocked
   — owner configured it); paste `gh api repos/Saber5656/debugdungeon/rulesets` output into the
   PR. Required status checks (`lint`, `test`): if absent, this issue does NOT mutate rulesets —
   it documents the exact UI/API change as a maintainer action item in repo-settings.md and the
   verify command that must pass afterwards.
5. Actions policy (settings): default workflow permissions = read-only; "Allow GitHub Actions to
   create and approve pull requests" = off:
   ```sh
   gh api -X PUT repos/Saber5656/debugdungeon/actions/permissions/workflow \
     -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
   gh api repos/Saber5656/debugdungeon/actions/permissions/workflow   # verify
   ```
6. Record final state: `docs/security/repo-settings.md` — table of control → state → verify command
   → date; audit (39) re-runs the verify column.

## Acceptance Criteria

- [ ] Dependabot opens (or would open — config validated by GitHub UI "recent updates" page) grouped PRs; config lint passes.
- [ ] CodeQL run green on main; alerts page active.
- [ ] Secret scanning + push protection + private vuln reporting: enabled, evidence via API outputs.
- [ ] Required status checks include lint+test; force-push blocked (API evidence).
- [ ] `docs/security/repo-settings.md` complete with verify commands that pass.

## Validation

All `--jq` verify commands return expected values (paste outputs in PR). Push-protection proof:
on a throwaway branch `test/push-protection`, commit a file containing the GitHub-documented
canary test secret pattern (a `ghp_`-prefixed 40-char dummy — NEVER a real credential), `git push`
→ paste the rejection message → delete the branch and the local commit. Steps + cleanup recorded
in repo-settings.md.

## Dependencies

02, 03 (SECURITY.md whose reporting channel this issue enables — its fallback line is removed here).

## Non-goals

OpenSSF Scorecard badge (optional follow-up), signed commits enforcement, CODEOWNERS (solo-maintainer; revisit with contributors).

## Design References

DESIGN §10.7; SECURITY.md (03); global git rules (no direct main pushes — already enforced).
