# ADR-004: Bundled scenarios ship via go:embed; images are tagged by scenario content hash

- Status: Accepted
- Deciders: Fable (design), reviewable by owner

## Context

Bundled content must reach players reliably (no "content dir not found"), and scenario images
must be rebuilt exactly when content changes, not more, not less.

## Decision

1. Bundled scenario directories are compiled into the binary with `go:embed`; the loader
   materializes a scenario's build context to a 0700 tmpdir only for `docker build`.
2. Every scenario gets a deterministic **content hash** (SHA-256 over sorted relative paths,
   a file-mode class bit, and file bytes; excluding `hints/`, `solution.md`, `solution.sh`).
   The local image tag is `debugdungeon/scn-<id>:<hash12>`; an existing tag short-circuits the build.

## Consequences

- One artifact to install; binary size stays small because rooms are recipes (text), not images.
- Cache correctness is structural: docs-only edits don't invalidate images; any Dockerfile/check
  change does. Upgrading the CLI silently upgrades rooms (new tags), leaving stale images for
  `clean --all` to purge (GC issue 16).
- Embedded FS must be treated read-only; authoring/community flows use real directories through
  the same loader interface (issues 10, 59).

## Alternatives considered

- **Content directory shipped beside the binary**: breaks under Homebrew relocation and manual
  copies; classic "works on my machine" failure.
- **Separate content download on first run**: violates zero-network posture (ADR-007) and adds
  an update/trust mechanism we don't need in v1.
- **`:latest`-style mutable tags**: cache invalidation bugs guaranteed.
