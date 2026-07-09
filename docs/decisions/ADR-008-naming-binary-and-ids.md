# ADR-008: Naming — binary `debugdungeon`, alias `ddgn`, kebab-case scenario IDs

- Status: Accepted
- Deciders: Fable (design), reviewable by owner (branding-sensitive)

## Context

The obvious short name `dd` collides with coreutils `dd` and must never be shipped. The binary
name appears in every doc, issue, and muscle memory.

## Decision

- Primary binary: **`debugdungeon`** (matches repo/product; tab-completion friendly).
- Convenience alias: **`ddgn`** installed as a symlink by release packaging (checked for collisions:
  no notable tool claims `ddgn` as of 2026-07; re-verify at issue 36, KU-8).
- Scenario IDs: `^[a-z0-9][a-z0-9-]{2,39}$`, stable forever once released (progress keys on them).
- Docker namespace: local tags `debugdungeon/scn-<id>:<hash12>`, labels `com.debugdungeon.*`.
- Homebrew formula: `debugdungeon` in tap `Saber5656/homebrew-tap`.

## Consequences

- No coreutils shadowing risk; docs always show `debugdungeon`, mentioning `ddgn` once.
- Renaming later would break progress files, labels, and muscle memory — treat as frozen after v1.0.0.
