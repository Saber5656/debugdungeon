# Title

CI workflow: lint + unit tests with hardened GitHub Actions

## Summary

Add a hardened GitHub Actions workflow that runs lint and unit tests on every PR and on pushes to
`main`, following the supply-chain rules in DESIGN §10.7.

## Context

This is the first workflow in the repo and sets the security pattern every later workflow
(scenarios 29, security 35, release 36, CodeQL 38) must copy: least-privilege permissions and
SHA-pinned actions.

## Scope

- `.github/workflows/ci.yml` only.
- Not in scope: Docker-dependent jobs (29, 34, 35), CodeQL (38), releases (36).

## Detailed Requirements

1. Triggers: `pull_request` and `push` to `main`. Add `paths-ignore: ["docs/**", "**.md"]` on both.
2. Top-level `permissions: contents: read`. No job may widen this.
3. `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
4. Jobs (all `runs-on: ubuntu-latest`, `timeout-minutes: 15`):
   - `lint`: checkout → setup-go (`go-version-file: go.mod`, cache enabled) → golangci-lint official action.
   - `test`: checkout → setup-go → `make test`.
5. Every third-party action referenced by **full commit SHA** with a trailing comment naming the
   version tag, e.g. `uses: actions/checkout@<sha> # v4.x.x`. Resolve current SHAs at
   implementation time; do not copy SHAs from this document.
6. No secrets consumed. No `pull_request_target`. No `actions/cache` custom keys beyond setup-go's builtin cache.
7. Both jobs must be green on a PR that only touches Go code, and skip on a docs-only PR.

## Acceptance Criteria

- [ ] PR touching Go code shows `lint` and `test` checks; both pass.
- [ ] Docs-only PR triggers neither job.
- [ ] `permissions` is exactly `contents: read` at workflow level; no job-level elevation.
- [ ] All `uses:` references are 40-char SHAs with version comments.
- [ ] Workflow has `concurrency` cancelation and job timeouts.

## Validation

Open a scratch PR with a whitespace Go change and a second with only a docs change; attach links
showing triggered/skipped runs. Run `actionlint` locally (or via `go run`) and paste clean output.

## Dependencies

01.

## Non-goals

Docker integration jobs, arm64 runners (29, KU-1), coverage upload, release automation.

## Design References

DESIGN §10.7, §12; ISSUE_PLAN §6.
