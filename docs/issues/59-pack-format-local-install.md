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
2. **Safe extraction** (tar.gz → temp dir 0700; rules BEFORE any validation):
   - Reject entry paths: absolute, containing `..`, or escaping root after Clean (zip-slip).
   - Reject: symlinks, hardlinks, devices, FIFOs — regular files + dirs only.
   - Strip setuid/setgid/sticky; files → 0600, dirs → 0700.
   - Quotas: ≤ 64 MiB total (compressed AND decompressed — count while streaming, abort over),
     ≤ 2,000 entries, path depth ≤ 8, member name rules per SV024.
   - Abort → temp dir removed, nothing installed.
3. **Install pipeline** `pack install <src>`:
   1. Detect dir vs tarball; tarball → safe-extract to temp.
   2. Parse+validate `pack.yaml`; enumerate scenario dirs; each: `LoadSpec` + `ValidateDir` (08 strict) — ANY violation → abort, report all.
   3. Collision policy: pack name already installed → require `--force` (replaces after confirm);
      scenario ID colliding with bundled or other installed packs → HARD abort (ErrIDCollision from 10; no override).
   4. Disclosure + confirm (minimal for this issue; 61 enriches): pack name/version/authors,
      scenario count/ids, total size, tmpfs/resource asks above defaults, base image FROM lines
      (parsed textually from Dockerfiles) → `Install pack '<name>'? Type the pack name to confirm:`
      (typed-name confirm; non-TTY → require `--yes-i-trust-this-pack`).
   5. Move into `<state>/packs/<name>/` (atomic: temp → rename within same volume).
   6. Write `packs/<name>/.provenance.json`:
      `{schema_version:1, name, version, source: {kind:"path"|"tarball", ref:<abs path>, sha256:<tarball hash|null>}, installed_at, cli_version, accepted_at: null}`
      (accepted_at consumed by 61's first-play gate).
7. Registry integration: `AddPackDir` per pack at startup (10); load failure of an installed pack
   → warn + skip (never crash the CLI; `pack list` shows it broken).
8. Attack corpus (committed tarballs built by test code, not binaries in git): path traversal,
   absolute path, symlink member, hardlink escape, setuid file, 100k-entries bomb, decompression
   bomb (small .gz → >64 MiB), colliding scenario id, hostile pack.yaml (unknown fields, bad name).

## Acceptance Criteria

- [ ] Every attack-corpus case: install aborts, temp cleaned, `packs/` untouched (table-tested).
- [ ] Happy path: template-based pack installs; `list` (27) shows its scenarios (source tag from 10); provenance file exact-schema golden-tested.
- [ ] Collision matrices: pack-name (needs --force), scenario-id (hard abort) — both covered.
- [ ] Non-TTY without the long flag → abort exit 1.
- [ ] Modes on installed tree: dirs 0700/files 0600 verified.

## Validation

`go test ./internal/pack/...` (attack corpus) + itest happy-path install → play a pack scenario end-to-end.

## Dependencies

05, 08, 10, 11.

## Non-goals

Git sources & uninstall/list commands (60), rich capability-diff UX + first-play gate (61), remote registries (v2, ADR-007).

## Design References

DESIGN §10.4 (pack row), §10.6; ADR-004 (same loader), ADR-007 (no fetching).
