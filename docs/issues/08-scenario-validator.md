# Title

Scenario validator: structural, semantic, and security rules + JSON Schema artifact

## Summary

Implement `scenario.Validate` enforcing every constraint in DESIGN §6.2/§10.4 over a loaded spec
*and* its directory tree, with stable rule codes and a negative-test corpus. Publish a
non-normative JSON Schema for editor tooling.

## Context

This is the primary defense at trust boundary TB3 (engine ↔ content) and the gate reused by
`scenario validate` (57) and pack install (59). Rules need stable codes so tests, docs, and CLI
output stay in sync.

## Scope

- `internal/scenario/validate.go`, `internal/scenario/rules.go` + corpus tests
- `docs/schemas/scenario-v1.schema.json` (documentation artifact)
- Not: content hashing (09), cookbook prose (12)

## Detailed Requirements

1. API: `Validate(fsys fs.FS, dir string, s *Spec) []RuleViolation` where
   `type RuleViolation struct { Rule string; Path string; Msg string }`. Empty slice = valid.
2. Rules (code → check). Implement exactly; codes are frozen API:
   - `SV001` schema_version == 1
   - `SV002` id matches `^[a-z0-9][a-z0-9-]{2,39}$` AND equals `path.Base(dir)`
   - `SV003` title non-empty, ≤ 60 chars (after trim)
   - `SV004` floor in 1..6; `SV005` difficulty in 1..5
   - `SV006` topics 1..6 items, each `^[a-z0-9-]{2,20}$`, unique
   - `SV007` time_estimate_min in 5..120
   - `SV008` lore non-empty, ≤ 1500 chars
   - `SV009` entry.shell == `/bin/bash` (only allowed value in schema v1, DESIGN §6.2); entry.workdir absolute path; entry.user `^[a-z_][a-z0-9_-]{0,31}$`
   - `SV010` build.context cleaned-relative, stays inside dir, exists, is a directory, contains `Dockerfile`
   - `SV011` locks 1..8; `SV012` lock ids unique, match `^[a-z0-9-]{2,32}$`
   - `SV013` each lock script is a relative path under `checks/`, exists, ≤ 64 KiB
   - `SV014` lock timeout_sec in 1..60
   - `SV015` hints 1..6; files under `hints/`, exist, each ≤ 4 KiB
   - `SV016` solution.walkthrough == `solution.md` and exists ≤ 64 KiB; `SV017` solution.script == `solution.sh` and exists ≤ 64 KiB
   - `SV018` resources within caps (memory_mb 64..2048, cpus 0.1..2.0, pids 16..1024)
   - `SV019` tmpfs: ≤ 2 mounts; each path absolute, not `/`, not under `/dungeon`, size_mb 1..256, total ≤ 512; nr_inodes 0 or 256..65536; paths pairwise distinct and non-nested (neither a path-prefix of the other)
   - `SV020` network == `none`
   - `SV021` **no symlinks anywhere** in the scenario tree (walk; any symlink is a violation)
   - `SV022` every file ≤ 4 MiB; total tree ≤ 16 MiB; ≤ 400 files; path depth ≤ 8
   - `SV023` all paths referenced in the manifest resolve (after `path.Clean`) inside the scenario dir; reject absolute or `..`-containing references
   - `SV024` file/dir names `^[A-Za-z0-9._-]+$` (portability)
   - `SV025` scenario dir must not contain entries named `.git`, `.github`, or files with setuid/setgid mode bits (check via fs.FileInfo when available; os-backed FS only — document that embed.FS strips modes)
3. Walking uses `fs.WalkDir` on the passed `fs.FS`; symlink detection requires an `os.DirFS`-backed
   root — expose `ValidateDir(osRoot string, s *Spec)` variant using `os.Lstat` for SV021/SV025;
   embedded content (already symlink-free by construction) uses the fs.FS path. Registry (10) and
   pack install (59) must call the strictest variant available.
4. Violations must be deterministic in order (sort by rule, then path).
5. `docs/schemas/scenario-v1.schema.json`: JSON Schema (draft 2020-12) mirroring §6.2 field
   constraints, header comment marking it **non-normative** (Go validator is normative).
6. Corpus: `testdata/badscenarios/<case>/` — one per rule above (≥ 25 cases) + 2 fully valid
   fixtures (minimal, maximal). A table test asserts exact rule codes per case.

## Acceptance Criteria

- [ ] Every rule SV001–SV025 has ≥ 1 failing corpus case asserting its code; valid fixtures return zero violations.
- [ ] Symlink escape attempt (`hints/01.md -> /etc/passwd`) fails SV021 via `ValidateDir`.
- [ ] Violation order deterministic (test shuffles input, output stable).
- [ ] JSON Schema file parses (any JSON parser) and documents the same numeric bounds (spot-checked in test for 3 fields).

## Validation

`go test ./internal/scenario/...` green; corpus tree committed; run `ValidateDir` against
`scenarios/_template` once issue 12 lands (noted as follow-up check there).

## Dependencies

07.

## Non-goals

Dockerfile content linting (cookbook/manual review; revisit post-v1), archive safety (59), schema-based runtime validation.

## Design References

DESIGN §6.2, §10.4 (TB3), §10.6; ADR-005 (network enum rationale).
