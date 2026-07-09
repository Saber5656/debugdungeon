# Title

Embedded scenario bundle and scenario registry

## Summary

Embed `scenarios/` into the binary with `go:embed` and implement the registry that loads, indexes,
and materializes scenarios from the embedded bundle (and, later, installed packs).

## Context

ADR-004: content ships inside the binary. Every game command resolves scenarios through this
registry; pack install (59) plugs into the same interface.

## Scope

- `scenarios/embed.go` (package `scenarios`, `//go:embed all:.` style embed of scenario dirs)
- `internal/scenario/registry.go` + tests
- Not: pack loading internals (59), image build (14)

## Detailed Requirements

1. Embed: `scenarios/embed.go` exposes `var FS embed.FS` embedding every scenario directory.
   Directories starting with `_` (e.g. `scenarios/_template`, issue 12) and `README.md` are
   **ignored by the registry** (still embeddable). Keep `scenarios/README.md` so the embed never
   fails while content is sparse.
2. `type Source struct { Kind string /* "bundled" | "pack" */; Pack string }`.
3. `type Loaded struct { Spec *scenario.Spec; Dir string; Source Source; FS fs.FS; Hash string }`.
4. `type Registry struct{ … }` with:
   - `LoadBundled() error` — enumerate top-level dirs of `scenarios.FS`, skip `_`-prefixed and
     non-dirs; for each: `LoadSpec` → `Validate` (fs.FS variant) → `ContentHash`; any invalid
     bundled scenario is a **fatal** registry error (bundled content must be perfect — CI enforces).
   - `AddPackDir(osRoot string, pack string) error` — same pipeline with `ValidateDir` (08); ID
     collision with an existing entry → error `ErrIDCollision` (used by 59; not wired to CLI yet).
   - `ByID(id string) (*Loaded, bool)`; `All() []*Loaded` sorted by (floor, difficulty, id);
     `Floors() [][] *Loaded` grouped.
   - `Materialize(ctx, l *Loaded) (buildCtxDir string, cleanup func(), err error)` — copy
     `l.Dir/image` (build context only, per DESIGN §7.2) to a fresh `os.MkdirTemp` dir with mode
     0700, preserving relative layout; files 0600 (exec bits irrelevant for build context — Dockerfile
     COPY sets its own). cleanup removes the tree; caller must always defer it.
5. Registry construction happens once per process in cli wiring; commands receive it via context
   accessor `cli.Registry(ctx)`.
6. Loading must NOT touch Docker or the network.

## Acceptance Criteria

- [ ] With a `testdata` fixture wired through an `fstest.MapFS`/embed test double: bundled load
  indexes valid scenarios, skips `_template`, hard-fails on one invalid scenario (error names the id + rule code).
- [ ] `ByID`, ordering of `All()` and `Floors()` covered by tests.
- [ ] `Materialize` produces a build context whose file set exactly equals `image/` (walk-compare test) and cleanup removes it.
- [ ] ID collision path returns `ErrIDCollision`.
- [ ] `go vet`/lint clean; no Docker/network imports in the package.

## Validation

`go test ./internal/scenario/... ./scenarios/...` green. Binary size check: `make build` and note
size delta in PR (< 1 MiB expected while content is sparse).

## Dependencies

07, 08, 09.

## Non-goals

Pack install UX (59–61), image building (14), lazy loading.

## Design References

DESIGN §4.2, §7.2; ADR-004.
