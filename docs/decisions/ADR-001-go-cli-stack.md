# ADR-001: Go + cobra + Docker SDK as the implementation stack

- Status: Accepted (owner decision, 2026-07-10)
- Deciders: Saber5656 (product owner), Fable (design)

## Context

DebugDungeon is a CLI game that orchestrates Docker: build images, create hardened containers,
run interactive TTY exec sessions, and pipe check scripts. It must install trivially on
macOS/Linux, start fast, and be implementable by lower-capability agents working from granular
issues. Candidates: Go, Rust, TypeScript/Node, Python.

## Decision

Implement in **Go 1.26.x** with:

- `spf13/cobra` for the command surface,
- `github.com/docker/docker/client` (official Engine API SDK, version negotiation),
- `gopkg.in/yaml.v3` (strict decoding) for scenario manifests,
- `charmbracelet/lipgloss` + `golang.org/x/term` for terminal presentation and raw-mode PTY handling,
- `go:embed` for bundled content (see ADR-004),
- goreleaser for multi-platform release artifacts.

`bubbletea` is deferred to Wave 9 (interactive map) to keep the MVP dependency set small.

## Consequences

- Single static binary per platform; Homebrew + GitHub Releases distribution is straightforward.
- The Docker SDK path (vs shelling out to `docker`) gives structured errors, no PATH trust issue,
  and testable interfaces — at the cost of a moderately large dependency tree (mitigated by
  `govulncheck` + Dependabot, DESIGN §10.7).
- Weaker implementation agents get a mainstream, heavily documented ecosystem (cobra/docker
  client examples are abundant), reducing guessing.

## Alternatives considered

- **Rust (bollard)**: excellent runtime properties; rejected because the Docker crate ecosystem is
  thinner and implementation difficulty is higher for mechanical execution by low-capability agents.
- **TypeScript/Node (dockerode)**: easy npm distribution but adds a runtime dependency, slower
  startup, and weaker single-binary story.
- **Python**: distribution friction (pipx) and startup latency for a game-feel CLI.
