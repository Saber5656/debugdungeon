# ADR-007: The CLI makes zero network calls; no telemetry in v1

- Status: Accepted
- Deciders: Saber5656 (product owner, via privacy posture), Fable (design)

## Context

Trust is a feature for a security-sensitive audience (the game literally runs containers on your
machine). Telemetry, update phone-home, or content downloads would each add network egress,
privacy questions, and release burden.

## Decision

The `debugdungeon` process itself performs **no network I/O**. The only network activity in the
system is the Docker daemon pulling pinned base images during `docker build`, which is documented
prominently. No telemetry, no crash upload, no update check in v1.

Wave 9 may add an **opt-in** (default off, config-gated) release check: single HTTPS GET to the
GitHub releases endpoint, no identifiers beyond the version User-Agent, result cached locally.
Anything beyond that requires a new ADR.

## Consequences

- Simple privacy story: "offline after first build"; easy security review; no data-handling policy needed.
- We lose usage insight (which rooms are too hard) — compensated post-v1 by GitHub Discussions
  templates asking for `stats` output voluntarily.
- Update discovery is manual (brew upgrade) until the opt-in check ships.
