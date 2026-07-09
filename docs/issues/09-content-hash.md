# Title

Deterministic scenario content hash

## Summary

Implement the content hash that keys scenario image tags and build caching (ADR-004):
SHA-256 over a deterministic walk of the scenario directory, excluding player-facing docs.

## Context

Image rebuild correctness depends on this hash: it must change iff build-relevant content changes.
DESIGN §7.2 defines tag `debugdungeon/scn-<id>:<hash12>`.

## Scope

- `internal/scenario/hash.go` + tests
- Not: image build (14)

## Detailed Requirements

1. `ContentHash(fsys fs.FS, dir string) (string, error)`:
   - Walk `dir` with `fs.WalkDir`, collect files, sort by slash-path ascending (byte order).
   - **Exclusions** (relative to scenario root): `hints/` (entire dir), `solution.md`, `solution.sh`.
     Everything else participates, including `scenario.yaml` and `checks/`.
   - Digest input per file, in order: `rel_path` + `\x00` + mode class byte (`d` dir, `x` file with
     any exec bit, `f` other file) + `\x00` + 8-byte big-endian length + file bytes. Directories
     contribute path+class only. (fs.FS without mode info → class `f`; document that bundled embed
     and os walks agree because template keeps scripts non-executable — cookbook rule, issue 12.)
   - Output: lowercase hex SHA-256; helper `Short(h string) string` returns first 12 chars.
2. Determinism requirements: no timestamps, no absolute paths, no map iteration order.
3. Golden vectors: commit a small `testdata/hashscn/` fixture and assert the exact hex digest in
   the test (so accidental algorithm changes fail loudly). Changing the algorithm later requires
   bumping the tag prefix (`scn-` → `scn2-`) — record this as a comment.
4. Property tests: (a) editing `hints/02.md` or `solution.sh` does NOT change the hash;
   (b) editing `image/Dockerfile` or `checks/x.sh` DOES; (c) renaming a file changes it;
   (d) chmod +x on a check changes it only on os-backed FS (documented).

## Acceptance Criteria

- [ ] Golden digest test passes and is byte-exact across macOS/Linux (CI of issue 02 proves linux).
- [ ] All four property tests pass.
- [ ] `Short()` returns 12 lowercase hex chars.

## Validation

`go test ./internal/scenario/... -run Hash` green on macOS (local) and ubuntu (CI).

## Dependencies

07.

## Non-goals

Cache eviction (16), remote content addressing.

## Design References

DESIGN §7.2; ADR-004, ADR-006 (why solution.sh is excluded).
