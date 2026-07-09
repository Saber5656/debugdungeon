# Title

Release pipeline: goreleaser, Homebrew tap, checksums, build provenance

## Summary

Implement the tag-triggered release pipeline producing signed-enough artifacts for four platforms,
a Homebrew formula with the `ddgn` alias, and supply-chain provenance (DESIGN §13, §10.7).

## Context

Distribution IS part of the security model (TB5): users must be able to verify what they run.
v1 ships checksums + GitHub artifact attestation; cosign deferred (KU-6).

## Scope

- `.goreleaser.yaml`, `.github/workflows/release.yml`
- Homebrew tap repo bootstrap instructions (`Saber5656/homebrew-tap`)
- Not: package managers beyond brew (apt/nix v2), auto-update (66)

## Detailed Requirements

1. `.goreleaser.yaml`:
   - builds: `darwin/arm64, darwin/amd64, linux/arm64, linux/amd64`; `CGO_ENABLED=0`; flags
     `-trimpath`; ldflags setting `internal/version.{Version,Commit,Date}`.
   - archives: tar.gz, name template `debugdungeon_<version>_<os>_<arch>`; include LICENSE+README.
   - `checksum: sha256` → `checksums.txt`.
   - brews: formula `debugdungeon` in tap `Saber5656/homebrew-tap`, install block installs binary
     AND `bin.install_symlink "debugdungeon" => "ddgn"`; test block runs `debugdungeon version`.
   - release notes: generated from commit log groups (feat/fix/scenario/docs).
2. **Pre-flight naming check (KU-8, ADR-008)**: document + execute at implementation:
   `brew search ddgn`, `brew search debugdungeon`, check formulae.brew.sh API — record results in
   the PR; conflict → escalate to owner before shipping (do not rename unilaterally).
3. `release.yml`:
   - Trigger: push of tag `v*`. `permissions: { contents: write, id-token: write, attestations: write }` (job-scoped).
   - Steps: checkout (fetch-depth 0) → setup-go → `make test` → goreleaser action (SHA-pinned) →
     `actions/attest-build-provenance` (SHA-pinned) over `dist/*.tar.gz` + `checksums.txt`.
   - Tap push token: repo secret `HOMEBREW_TAP_TOKEN` — **created and stored manually by the
     maintainer** (fine-grained PAT, contents:write on the tap repo only). The issue documents the
     exact PAT scopes; the agent must NOT create or handle the token value (handoff step).
4. Snapshot job: on PRs touching `.goreleaser.yaml` or `release.yml`, run `goreleaser release
   --snapshot --clean` and `dist/*/debugdungeon version` smoke (linux binaries via container).
5. Tap repo: document one-time creation (`gh repo create Saber5656/homebrew-tap --public` + README)
   as a maintainer step in the PR body; goreleaser manages formula content thereafter.
6. Install doc snippet (final text lands in 37): brew install, tarball+checksum verify one-liner,
   `go install` fallback, attestation verify command (`gh attestation verify`).

## Acceptance Criteria

- [ ] Snapshot build produces 4 working binaries (`version` output correct per platform — linux ones smoke-run in docker).
- [ ] Tag `v0.1.0-rc1` on a fork/test run produces: GitHub release with 4 archives + checksums + attestation; formula lands in tap; `brew install saber5656/tap/debugdungeon` then `debugdungeon version` + `ddgn version` work on macOS (manual evidence).
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
