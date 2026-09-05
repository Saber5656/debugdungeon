# Title

Scenario pack format and local install (safe extraction)

## Summary

Define the community pack format (`pack.yaml` + scenario dirs) and implement
`debugdungeon pack install <path|tarball>` with hostile-archive-safe extraction, full per-scenario
validation, collision policy, and provenance recording.

## Context

This opens trust boundary TB4 (DESIGN §10.6): community content is UNTRUSTED input. Extraction and
validation here are security-critical; the consent UX layers on in 61.

## Scope

- `internal/pack/format.go`, `internal/pack/extract.go`, `internal/pack/install.go`, `internal/cli/pack_install.go` + attack-corpus tests
- Not: git installs (60), trust-gate UX polish (61 — this issue ships a minimal y/N confirm)

## Detailed Requirements

1. **Pack format**: a directory (or tar.gz of one):
   ```
   <pack-root>/
   ├── pack.yaml          # {schema_version: 1, name: ^[a-z0-9][a-z0-9-]{2,29}$, version: semver,
   │                      #  description: ≤200 chars, authors: [strings]}
   └── <scenario-id>/…    # 1–20 scenario dirs, standard layout
   ```
   Strict YAML decode (07 pattern); unknown fields rejected.
2. **Safe extraction** (tar.gz → temp dir 0700; rules BEFORE any validation).
   Archive root shape: EITHER `pack.yaml` at the archive root, OR exactly one top-level directory
   containing it (that directory is stripped); anything else → reject with a shape message.
   - Reject entry paths: absolute, containing `..`, or escaping root after Clean (zip-slip).
   - Reject: symlinks, hardlinks, devices, FIFOs — regular files + dirs only.
   - Strip setuid/setgid/sticky; files → 0600, dirs → 0700.
   - Quotas: ≤ 64 MiB total (compressed AND decompressed — count while streaming, abort over),
     ≤ 2,000 entries, path depth ≤ 8, member name rules per SV024.
   - Abort → temp dir removed, nothing installed.
3. **Install pipeline** `pack install <src>`:
   1. Detect dir vs tarball; tarball → safe-extract to temp. **Directory sources get the SAME
      tree gate as archives** (review catch — no bypass): full `Lstat` walk rejecting
      symlinks/hardlinks/devices/FIFOs/setuid-setgid bits, same quotas (≤ 64 MiB, ≤ 2,000 entries,
      depth ≤ 8, SV024 names), then a normalizing COPY into the temp staging dir (files 0600,
      dirs 0700) — install always proceeds from the staged copy, never the original.
   2. Parse+validate `pack.yaml`; enumerate scenario dirs; each: `LoadSpec` + `ValidateDir` (08 strict) — ANY violation → abort, report all.
   3. Collision policy: pack name already installed → require `--force`, which replaces via:
      stage new pack fully → rename old dir to sibling `.trash-<ts>` → rename staged into place →
      on any failure, rename `.trash-<ts>` back (rollback) → success: RemoveAll the trash (a
      leftover `.trash-*` after a crash is swept at next CLI start — same sweep as 60's remove).
      Scenario ID colliding with bundled or other installed packs → HARD abort (ErrIDCollision
      from 10; no override).
   4. Disclosure + confirm (minimal for this issue; 61 enriches): pack name/version/authors,
      scenario count/ids, total size, tmpfs/resource asks above defaults, base image FROM lines
      (parsed textually from Dockerfiles), and one fixed warning line — installing allows this
      pack's Dockerfiles to run at first play in the Docker build environment, which has network
      access (DESIGN §10.6) → `Install pack '<name>'? Type the pack name to confirm:`
      (typed-name confirm; non-TTY → require `--yes-i-trust-this-pack`).
   5. Write `.provenance.json` INTO the staged copy and fsync it BEFORE the final move (re-review
      catch: writing it after the rename risks an installed pack with no provenance/accepted_at):
      `{schema_version:1, name, version, source: {kind:"path"|"tarball", ref:<abs path>, sha256:<tarball hash|null>}, installed_at, cli_version, accepted_at: null}`
      (accepted_at consumed by 61's first-play gate). If provenance write/fsync fails → abort and
      clean the staging dir before touching any existing pack.
   6. Move staged pack into `<state>/packs/<name>/` (atomic rename within the same volume; for
      `--force` replacement use the trash-swap sequence in step 3).
7. Registry integration: `AddPackDir` per pack at startup (10); load failure of an installed pack
   → warn + skip, recording the reason for 60's `pack list` to display (the `pack list` COMMAND
   itself is issue 60's scope — this issue only ensures the skip-with-reason data exists).
8. Attack corpus (committed tarballs built by test code, not binaries in git): path traversal,
   absolute path, symlink member, hardlink escape, setuid file, 100k-entries bomb, decompression
   bomb (small .gz → >64 MiB), colliding scenario id, hostile pack.yaml (unknown fields, bad name).

## Acceptance Criteria

- [ ] Every attack-corpus case: install aborts, temp cleaned, `packs/` untouched (table-tested).
- [ ] Happy path: template-based pack installs; REGISTRY-level assertion (`Registry.ByID` finds the pack scenario with Source pack:<name>) — CLI `list` rendering and end-to-end play are 61's validation, not this issue's; provenance file exact-schema golden-tested.
- [ ] Collision matrices: pack-name (needs --force), scenario-id (hard abort) — both covered.
- [ ] Non-TTY without the long flag → abort exit 1.
- [ ] Modes on installed tree: dirs 0700/files 0600 verified.

## Validation

`go test ./internal/pack/...` (attack corpus) + itest happy-path install → registry lookup.
(End-to-end play of a pack scenario happens in 61's journey itest, after the first-play gate exists.)

## Dependencies

05, 08, 10, 11.

## Non-goals

Git sources & uninstall/list commands (60), rich capability-diff UX + first-play gate (61), remote registries (v2, ADR-007).

## Design References

DESIGN §10.4 (pack row), §10.6; ADR-004 (same loader), ADR-007 (no fetching).
