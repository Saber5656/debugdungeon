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
   - Walk `dir` with `fs.WalkDir`; digest paths are slash-separated, `path.Clean`ed, **relative to
     the scenario root** (never containing `dir` itself); the root entry `.` is excluded.
   - **Exclusions** (relative to scenario root): `hints/` (the directory entry AND all descendants —
     return `fs.SkipDir` on it), `solution.md`, `solution.sh`. Everything else participates,
     including `scenario.yaml` and `checks/`.
   - Entries sorted by relative slash-path ascending (byte order). Record encoding:
     - directory: `rel_path` + `\x00` + `d` + `\x00`
     - file: `rel_path` + `\x00` + class (`x` any exec bit, else `f`) + `\x00` + 8-byte big-endian
       length + file bytes
     (fs.FS without mode info → class `f`; bundled embed and os walks agree because the cookbook
     keeps scripts non-executable — issue 12.)
   - Non-regular entries (symlinks, devices, sockets — only possible on os-backed FS) → error
     (callers validate first per issue 08 SV021; the hash refuses rather than guesses). Read/stat
     failures → error (never a partial hash).
   - Output: lowercase hex SHA-256; helper `Short(h string) string` returns the first 12 chars and
     panics on input shorter than 64 hex chars (programming error, not runtime input).
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
