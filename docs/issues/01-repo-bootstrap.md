# Title

Bootstrap Go module, repository layout, Makefile, and linting

## Summary

Create the Go module `github.com/Saber5656/debugdungeon`, the canonical directory layout, a
Makefile with the standard targets, and golangci-lint configuration, so every later issue lands
into a consistent skeleton.

## Context

The repository currently contains only `README.md` and `docs/`. All engineering issues assume the
module path, directory names, and make targets defined here. Toolchain: Go 1.26.x (DESIGN §4.3).

## Scope

- `go.mod` (`go.sum` arrives with the first dependency, issue 04)
- Directory skeleton and placeholder `main.go`
- `Makefile`, `.golangci.yml`, `.gitignore`, `.editorconfig`
- No CI (issue 02), no real commands (issue 04)

## Detailed Requirements

1. `go mod init github.com/Saber5656/debugdungeon`; `go.mod` contains `go 1.26`.
2. Create directories (empty packages get a `doc.go` with a one-line package comment):
   - `cmd/debugdungeon/main.go` — prints `debugdungeon (dev)` and exits 0 for now.
   - `internal/` — created on demand by later issues; do NOT pre-create empty subpackages other
     than `internal/version/version.go` exposing `var (Version = "dev"; Commit = "none"; Date = "unknown")`.
   - `scenarios/README.md` — one paragraph: bundled scenario content lives here (content arrives in issue 30+).
3. `Makefile` targets (all must work on macOS and Linux, bash-free POSIX where possible):
   - `build` → `mkdir -p bin` then
     `go build -trimpath -ldflags "-s -w -X $(MODULE)/internal/version.Version=$(VERSION) -X $(MODULE)/internal/version.Commit=$(COMMIT) -X $(MODULE)/internal/version.Date=$(DATE)" -o bin/debugdungeon ./cmd/debugdungeon`
     with `MODULE := github.com/Saber5656/debugdungeon`, `VERSION ?= dev`,
     `COMMIT ?= $(shell git rev-parse --short HEAD)`, `DATE ?= $(shell date -u +%Y-%m-%dT%H:%M:%SZ)`.
   - `main.go` imports `internal/version` and prints exactly `debugdungeon <version.Version>\n` (so the ldflags are exercised).
   - `test` → `go test -race -count=1 ./...`
   - `itest` → `go test -race -count=1 -tags=itest ./...` (Docker-dependent tests use build tag `itest`)
   - `lint` → `golangci-lint run`
   - `fmt` → `gofmt -l -w .` (fail-free)
   - `clean` → remove `bin/`
4. `.golangci.yml`: enable `govet, staticcheck, errcheck, revive, gosec, misspell, unconvert, gocritic`;
   gosec excludes may be added per-finding with `#nosec` + justification comment only.
   golangci-lint version: pin the then-current stable (per CONVENTIONS C6) in a Makefile comment;
   `make lint` fails with an install hint (`brew install golangci-lint` / official script) when absent.
5. `.gitignore`: `bin/`, `dist/`, `*.test`, `coverage.out`, `coverage.html`, `*.coverprofile`, `.DS_Store`.
   `go.sum` note: absent until the first external dependency (issue 04 adds cobra); commit it when generated.
6. `.editorconfig`: tabs for `.go`, 2-space for YAML/MD, final newline.
7. Add one trivial test (`internal/version/version_test.go`) asserting defaults, so `make test` exercises the harness.

## Acceptance Criteria

- [ ] `make build` produces `bin/debugdungeon`; running it prints a line containing `debugdungeon` and exits 0.
- [ ] `make test`, `make lint`, `make fmt` all exit 0 on a clean checkout.
- [ ] `go.mod` module path and `go 1.26` directive exactly as specified.
- [ ] No directories or files outside the list above are introduced.

## Validation

Run `make build test lint` locally on macOS (arm64) and in a `golang:1.26` container (amd64);
paste both outputs into the PR description.

## Dependencies

None.

## Non-goals

CI workflows (02), cobra wiring (04), release ldflags automation (36).

## Design References

DESIGN §4.2 (module map), §4.3 (toolchain); ADR-001.
