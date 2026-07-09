# ADR-006: Every scenario ships solution.sh; CI replays it as a solvability gate

- Status: Accepted
- Deciders: Fable (design), reviewable by owner

## Context

The #1 content quality risk is an unsolvable or trivially-open room: a lock that can never open,
breakage that didn't apply, or a fix that works on amd64 but not arm64. Human playtesting doesn't
scale to 20+ rooms × 2 architectures × every refactor.

## Decision

`solution.sh` (machine-executable intended fix) is a **required** scenario artifact. CI and
`scenario test` enforce, per scenario and per architecture:

1. Build image; create container with the production security profile.
2. Assert **at least one lock is CLOSED** on the pristine room (breakage exists).
3. Run `solution.sh` (root, stdin-piped, 300s budget).
4. Assert **all locks OPEN** (room is solvable exactly as shipped).

## Consequences

- Solvability, breakage presence, arch neutrality, and profile compatibility become regression-
  tested invariants instead of review hopes; content PRs from the community get the same gate.
- Authors must express the fix as a script — usually easy, occasionally constraining
  (interactive-only fixes are effectively banned; that's acceptable for check-verifiable rooms).
- `solution.sh` is spoiler material: excluded from images and from the content hash, never
  printed by the game (DESIGN §6.4).
- CI needs real Docker and an arm64 runner (KU-1); cost bounded by path-filtering + weekly full runs.
