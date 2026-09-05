# Issue Reading Conventions (normative for all docs/issues/NN-*.md)

Every implementation issue in this directory is read under these conventions. They resolve the
recurring "how exact is exact?" questions so individual issues don't repeat them. Where an issue
contradicts these conventions, the issue wins for its own scope.

## C1. API signatures

Function/type signatures quoted in issues are **normative for names, parameters' meaning, and
behavior; illustrative for exact Go types**. The implementer finalizes idiomatic signatures
(context placement, pointer-ness, option structs) in code review. When an issue references a type
owned by another issue (e.g. `scenario.Loaded`, `locks.Report`), the owning issue's definition is
authoritative — do not redeclare.

## C2. The `dockerx.API` interface

`dockerx.API` (issue 13) grows incrementally: each issue that needs a new Docker SDK call adds the
exact method it uses (same shape as the SDK method, minus the client receiver) plus a mock in the
shared test fakes. Needing a method not yet on the interface is expected, not a spec gap.

## C3. Scenario issues — manifest and file layout defaults

A scenario issue specifies breakage, locks, hints, and solution **content**. Everything else is
mechanical:

- `scenario.yaml` starts from `scenarios/_template/scenario.yaml` and applies the issue's manifest
  values (id, floor, difficulty, topics, time_estimate_min, resources/mounts when stated).
  Unstated fields take schema defaults (DESIGN §6.2). `title` and `lore` are authored to fit the
  issue's room description (lore ≤ 1500 chars, sanitization-safe ASCII).
- Lock scripts live at `checks/<lock-id>.sh`, one per lock, listed in the manifest in the issue's
  lock order. Every lock sets an explicit `timeout_sec` (≤ 60); when the issue doesn't state one,
  use 30 for retry-loop checks and 10 otherwise. Retry loops must budget ≤ `timeout_sec − 5s`.
- Hints are `hints/01.md … NN.md` in escalation order; `solution.walkthrough: solution.md`,
  `solution.script: solution.sh`.
- Dockerfile preamble is always the cookbook pattern:
  `ARG BASE=debian:bookworm-slim@sha256:<digest from COOKBOOK §2>` + `FROM ${BASE}`; apt installs
  use `--no-install-recommends` and clean `/var/lib/apt/lists/*`; never COPY hints/checks/solution
  into the image; keep bash + coreutils (incl. `timeout`) installed; amd64+arm64 compatible.
- Boot contract: `ENTRYPOINT ["/dungeon-boot.sh"]`; the script performs scenario setup
  (including ALL content under tmpfs mounts), starts services/loops, then
  `mkdir -p /var/dungeon && touch /var/dungeon/boot-ok` **last** before `exec sleep infinity`.
- Check scripts are deterministic POSIX sh, run as root via stdin-pipe under in-container
  `timeout` (DESIGN §7.7); failure messages use the `MSG:` convention.

## C4. Dependencies

`Dependencies` lists issues whose merged artifacts this issue consumes. A parenthesized entry
means "required only for the named sub-feature". `docs/ISSUE_PLAN.md` §3 is the normative table;
if an issue body and §3 disagree, §3 wins.

## C5. Validation phrasing

- `make test` / `make itest` / `make e2e` / `make itest-scenarios DD_SCENARIO_FILTER=<id>` are the
  canonical commands (issues 01/29/34 define them).
- "Transcript in PR" = a copy-paste (or `script(1)` capture) of the exact commands and outputs that
  demonstrate the named Acceptance Criteria, attached to the PR description. ANSI may be stripped.
- "Both arches" = the CI matrix of issue 29 (or a documented local run on the other arch).

## C6. Version and digest placeholders

`<sha>`, `<digest>`, `@vX.Y.Z`, `<ver>` placeholders are resolved at implementation time to the
then-current stable release (verified from the official source), recorded in the implementing PR.
Pinned versions live where the artifact lives (workflow file, Makefile, COOKBOOK §2) — never
copied from issue text as-is.

## C7. Sanitization default

Every string originating from scenario content, pack metadata, or container output that is
rendered anywhere except the raw PTY session MUST pass `textsafe.Sanitize`/`SanitizeInline`
(issue 11) with a stated or sensible cap (titles 60, ids 40, messages 200, bodies 4 KiB). Issues
name the load-bearing cases; unnamed scenario-sourced strings are NOT exempt.

## C8. Exit codes and errors

DESIGN §5.3 is the single exit-code contract. Typed/sentinel errors named in issues (e.g.
`ErrBusy`, `ErrIDCollision`) are exported from the owning package; consumers match with
`errors.Is/As`.

## C9. State-file writes

All state writes follow DESIGN §8.4: temp file (0600) in the same dir → fsync → atomic rename;
dirs 0700; corrupt files quarantined as `<name>.corrupt-<unix-ts>`; `schema_version` greater than
supported → typed "upgrade debugdungeon" error, file untouched.

## C10. Tool pinning in CI

Every workflow follows issue 02's pattern: third-party actions pinned to full 40-char commit SHAs
with version comments, least-privilege `permissions:`, timeouts, and concurrency groups. Tool
binaries invoked in CI (golangci-lint, goreleaser, govulncheck) are version-pinned in the workflow
or Makefile per C6.
