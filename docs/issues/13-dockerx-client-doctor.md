# Title

Docker client wrapper and `doctor` command

## Summary

Implement `internal/dockerx` (client init, capability checks, typed errors) and the user-facing
`debugdungeon doctor` diagnostics command.

## Context

Docker is the hard prerequisite (ADR-002); DESIGN §11 F1/F2/F12 route through this layer. Doctor
quality is a product feature (research doc, risk 3).

## Scope

- `internal/dockerx/client.go`, `internal/dockerx/errors.go`
- `internal/cli/doctor.go`
- Not: build (14), lifecycle (15), GC listing beyond read-only counts (16)

## Detailed Requirements

1. `dockerx.New(ctx) (*Client, error)`: `client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())`.
   `*Client` wraps the SDK client and exposes only the methods later issues need (interface-first
   for mocking: define `type API interface` with the calls used repo-wide; grows in 14/15/17/19).
2. `Preflight(ctx) (Facts, error)` runs, in order, mapping failures to typed errors:
   - `Ping` → `ErrUnreachable` (wrap OS-specific start hints: darwin "start Docker Desktop or OrbStack";
     linux "systemctl start docker / check $DOCKER_HOST"; include original error detail for `--verbose`).
   - negotiated API version < **1.44** → `ErrAPITooOld{Found, Want}`.
   - `Info`: `OSType != "linux"` → `ErrNotLinuxDaemon` (F12).
   - Collect `Facts{ServerVersion, APIVersion, OSType, Arch, NCPU, MemTotal, DockerRootDir, Endpoint}`.
3. Typed errors carry `exitcode.DockerUnavailable (3)` via `exitcode.Coded`.
4. `doctor` command output (stdout, table via simple fmt — ui package may not exist yet):
   - Docker: reachable? endpoint **redacted** (print scheme + host/socket path only; strip
     userinfo and query strings from `DOCKER_HOST`-style URLs before any output/log — TB1
     credential hygiene), server/API version, OSType/arch, CPUs, memory.
   - DebugDungeon: version, state root path, state-dir status: missing dirs → `warn` (doctor
     creates NOTHING); present → writable probe (create+delete `.probe` file); config file found?
   - Managed leftovers: count of containers and images labeled `com.debugdungeon.managed=true`
     (read-only `ContainerList`/`ImageList` with label filter; zero is fine) + total managed image
     size via the `DiskUsage` API (fallback: sum of `ImageSummary.Size` — shared layers may
     double-count; print `~` prefix); suggest `clean` when > 0 / > 10 GiB. Listing failure after a
     healthy ping → `warn`, not FAIL.
   - Each line prefixed `ok` / `warn` / `FAIL`.
5. Exit-code precedence: any Docker-section FAIL → 3 (wins over everything); else any
   state-dir FAIL (unwritable existing dir) → 1; warnings only → 0.
6. `doctor` must complete < 5s (context timeout per call: 3s ping, 5s总). On unreachable daemon it
   still prints the DebugDungeon section (partial report).

## Acceptance Criteria

- [ ] Unit tests with a mocked `API`: healthy path, unreachable, old API, windows OSType — assert messages + exit codes.
- [ ] `doctor` on a machine with Docker running prints all `ok` and exits 0 (manual transcript in PR).
- [ ] `doctor` with `DOCKER_HOST=tcp://127.0.0.1:1` prints FAIL row + per-OS hint, exits 3, and still shows state-dir section.
- [ ] No command other than the probe writes anything (probe file cleaned up).

## Validation

`go test ./internal/dockerx/... ./internal/cli/...` (doctor command paths: healthy, unreachable,
old API, non-linux OSType, unwritable state dir — mocked API, asserting messages + exit codes);
manual transcripts (healthy + broken DOCKER_HOST) attached.
Integration smoke behind `itest` tag: `Preflight` against real daemon.

## Dependencies

04, 05, 06.

## Non-goals

Podman-specific handling (KU-2 — only ensure errors are not misleading), disk-space math beyond Info fields, network diagnostics.

## Design References

DESIGN §4.3, §7.1, §11 (F1, F2, F12); ADR-002.
