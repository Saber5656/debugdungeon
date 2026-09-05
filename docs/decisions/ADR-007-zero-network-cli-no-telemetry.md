# ADR-007: No implicit network I/O from the CLI; no telemetry in v1

- Status: Accepted
- Deciders: Saber5656 (product owner, via privacy posture), Fable (design)

## Context

Trust is a feature for a security-sensitive audience (the game literally runs containers on your
machine). Telemetry, update phone-home, or content downloads would each add network egress,
privacy questions, and release burden.

## Decision

The `debugdungeon` process performs **no implicit network I/O**. Every network operation in the
system is user-initiated and enumerable (DESIGN §10.8):

1. The Docker daemon pulling pinned base images during `docker build` (triggered by the user's
   `play` / `scenario test`) — documented prominently.
2. `pack install <git-url>` invoking the system `git` at the user's explicit command (Wave 8).
3. The Wave 9 **opt-in** (default off, config-gated) release check: single HTTPS GET to the GitHub
   releases endpoint, ≤ 1/24h, no identifiers beyond the version User-Agent, result cached locally.

No telemetry, no crash upload, no implicit update checks — ever without a new ADR. In the MVP
(v1.0.0, Waves 0–5) none of the carve-outs beyond (1) exist, so the MVP CLI is strictly
zero-egress.

## Consequences

- Simple privacy story: "offline after first build"; easy security review; no data-handling policy needed.
- We lose usage insight (which rooms are too hard) — compensated post-v1 by GitHub Discussions
  templates asking for `stats` output voluntarily.
- Update discovery is manual (brew upgrade) until the opt-in check ships.
