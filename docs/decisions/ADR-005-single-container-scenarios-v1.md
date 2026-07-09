# ADR-005: v1 scenarios are single-container only (no compose topologies)

- Status: Accepted
- Deciders: Fable (design), reviewable by owner

## Context

Many attractive faults are distributed (app ↔ db across a network). Multi-container scenarios
would require engine-managed networks, startup ordering, readiness, multi-image builds, and a
much larger security/GC surface.

## Decision

v1 scenarios are **exactly one container** with `network: none`. Distributed-feeling faults are
modeled on localhost inside the container (e.g., nginx → local backend, app → local PostgreSQL),
which Floors 4–5 prove out. The schema's `network` field is a closed enum (`none`) so a future
`internal` value can be introduced via schema_version bump without breaking old CLIs.

## Consequences

- Engine lifecycle, GC, `doctor`, and the security profile stay simple and auditable for MVP.
- Content authors lose true network debugging (routing, DNS servers, firewalls) — partially
  compensated by hosts/nsswitch-class puzzles; honest cookbook section lists what's infeasible.
- Revisit trigger: first concrete v2 design for `network: internal` (engine-created, `internal=true`
  bridge shared by scenario-defined sidecars) must come with its own threat-model delta.
