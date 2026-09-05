# Title

Release pipeline: goreleaser, Homebrew tap, checksums, build provenance

## Summary

Implement the tag-triggered release pipeline producing signed-enough artifacts for four platforms,
a Homebrew formula with the `ddgn` alias, and supply-chain provenance (DESIGN §13, §10.7).

## Context

Distribution IS part of the security model (TB5): users must be able to verify what they run.
v1 ships checksums + GitHub artifact attestation; cosign deferred (KU-6).

## Scope

- `.goreleaser.yaml`, `.github/workflows/release.yml`, `docs/RELEASING.md`
- Homebrew tap repo bootstrap instructions (`Saber5656/homebrew-tap`)
- Not: package managers beyond brew (apt/nix v2), auto-update (66)

## Detailed Requirements

1. `.goreleaser.yaml`:
   - builds: `darwin/arm64, darwin/amd64, linux/arm64, linux/amd64`; `CGO_ENABLED=0`; flags
     `-trimpath`; ldflags setting `internal/version.{Version,Commit,Date}`.
   - archives: tar.gz, name template `debugdungeon_<version>_<os>_<arch>`; include LICENSE+README.
   - `checksum: sha256` → `checksums.txt`.
   - brews: formula `debugdungeon` in tap `Saber5656/homebrew-tap` with
     `repository: { owner: Saber5656, name: homebrew-tap, token: "{{ .Env.HOMEBREW_TAP_TOKEN }}" }`;
     install block installs binary AND `bin.install_symlink "debugdungeon" => "ddgn"`; test block
     runs `debugdungeon version`.
   - changelog: `changelog.groups` config — order: `feat` → `fix` → `scn|scenario` → `docs` →
     catch-all; `filters.exclude: ["^chore", "^ci"]` (exact regexes finalized at implementation,
     committed in `.goreleaser.yaml`).
2. **Pre-flight naming check (KU-8, ADR-008)**: document + execute at implementation:
   `brew search ddgn`, `brew search debugdungeon`, check formulae.brew.sh API — record results in
   the PR; conflict → escalate to owner before shipping (do not rename unilaterally).
3. `.github/workflows/release.yml` — ONE workflow, two explicitly-guarded jobs (no implicit event mixing):
   - `on: { push: { tags: ["v*"] }, pull_request: { paths: [".goreleaser.yaml", ".github/workflows/release.yml"] } }`.
   - Job `release` (`if: github.event_name == 'push'`):
     `permissions: { contents: write, id-token: write, attestations: write }` (job-scoped);
     checkout (fetch-depth 0) → setup-go → `make test` → goreleaser action (SHA-pinned, and the
     goreleaser BINARY version pinned via the action's `version:` input per C6) →
     `actions/attest-build-provenance` (SHA-pinned) over `dist/*.tar.gz` + `checksums.txt`.
   - Job `snapshot` (`if: github.event_name == 'pull_request'`): `permissions: contents: read`;
     `goreleaser release --snapshot --clean` → smoke: run BOTH linux binaries via docker
     (`--platform` amd64 + arm64 w/ qemu setup action) asserting `version` output; darwin archives
     are verified structurally only (`tar -tzf` contains the binary + LICENSE) — executing darwin
     binaries needs the macOS rc-flow below.
   - Tap push token: repo secret `HOMEBREW_TAP_TOKEN` — **created and stored manually by the
     maintainer** (fine-grained PAT: repository access = `Saber5656/homebrew-tap` only, permission
     = Contents read/write). The agent must NOT create or handle the token value (handoff step).
5. Tap repo + operational docs live in `docs/RELEASING.md` (new file, this issue): one-time tap
   creation (`gh repo create Saber5656/homebrew-tap --public` + README), PAT scopes + secret
   registration steps (maintainer-manual), rc-tag rehearsal procedure, tag → release runbook,
   rollback notes.
6. Install doc snippet (final text lands in 37): brew install, tarball+checksum verify one-liner,
   `go install` fallback, attestation verify command (`gh attestation verify`).

## Acceptance Criteria

- [ ] Snapshot job: 4 archives; both linux binaries smoke-run (amd64 + arm64/qemu); darwin archives structurally verified.
- [ ] Rc rehearsal ON THIS REPO (forks lack the tap secret — documented in RELEASING.md): push tag
  `v0.1.0-rc.1` marked prerelease → GitHub release with 4 archives + checksums + attestation;
  formula lands in tap; `brew install saber5656/tap/debugdungeon` then `debugdungeon version` +
  `ddgn version` work on the maintainer's macOS (manual evidence); rc release + tag then deleted
  per the RELEASING.md cleanup step (formula left pointing at rc until first real release — noted).
- [ ] checksums.txt matches shipped archives (manual re-hash evidence).
- [ ] Naming pre-flight results recorded; no unresolved collision.
- [ ] Workflow permissions job-scoped exactly as listed; all actions SHA-pinned.

## Validation

rc-tag dry-run evidence bundle in the PR (links, transcripts). `make test` unaffected.

## Dependencies

01, 02, 04.

## Non-goals

cosign (KU-6/v2), Windows builds (KU-3), nightly builds, auto-bumping deps.

## Design References

DESIGN §13, §10.7 (TB5); ADR-001, ADR-007, ADR-008; KU-6, KU-8.
