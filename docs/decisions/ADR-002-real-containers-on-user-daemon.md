# ADR-002: Rooms are real containers on the user's Docker daemon, under a fixed hardened profile

- Status: Accepted (owner decision, 2026-07-10)
- Deciders: Saber5656 (product owner), Fable (design)

## Context

The core promise is "repair intentionally broken environments". Options for what a room is:
(a) a real container on the player's Docker-compatible daemon, (b) broken code + failing tests,
(c) an in-process simulated shell/filesystem, (d) remote VMs we host.

## Decision

**(a) Real Linux containers created by the CLI on the player's own Docker daemon**, with a
**fixed, non-scenario-overridable security profile** (DESIGN §7.4/§10.3): network `none`,
CapDrop ALL + minimal allowlist, no-new-privileges, resource clamps (memory/cpu/pids/tmpfs),
no host mounts/devices/namespaces, never privileged, `Init: true`.

Docker Engine API is the integration point (Docker Desktop / OrbStack / colima / rootless moby;
Podman best-effort). Scenarios must be authorable within the fixed profile; puzzle classes that
require extra capabilities (iptables, mount remount, immutable attrs) are declared infeasible in
the cookbook rather than weakening the profile.

## Consequences

- Maximum realism and skill transfer; the engine stays thin (content is data).
- Docker becomes a hard prerequisite → `doctor` command and first-run UX must be excellent;
  documented in DESIGN §11 failure modes.
- Security architecture concentrates on one boundary (host ↔ container) that Docker already
  hardens; our job is to never loosen defaults and to regression-test the profile (issue 35).
- Base images must be pulled once (network needed at first build) — offline-after-first-build.

## Alternatives considered

- **Simulated shell (c)**: zero Docker dependency and browser-portable, but a huge bespoke engine,
  low realism, and endless "the fake shell doesn't support X" issues. Rejected for v1; remains a
  v2 idea (multi-engine abstraction explicitly out of scope, DESIGN §2.3).
- **Broken code + tests (b)**: a different product (kata trainer); rejected as primary concept.
- **Hosted VMs (d)**: violates local-first/no-server posture and adds operating costs and
  multi-tenant security burden.
