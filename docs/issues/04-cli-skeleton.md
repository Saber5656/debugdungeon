# Title

cobra CLI skeleton: root command, global flags, exit-code contract, version, completion

## Summary

Wire cobra into `cmd/debugdungeon`, implement the process exit-code contract (DESIGN §5.3), global
flags, `version` and `completion` commands, and a panic trap. Every later command plugs into this
skeleton.

## Context

The exit-code table is a CLI-wide API used by tests (34) and scripts. Errors must map to codes via
typed errors, not scattered `os.Exit` calls.

## Scope

- `internal/cli/root.go`, `internal/cli/version.go`
- `internal/exitcode/exitcode.go`
- `cmd/debugdungeon/main.go` rewrite
- Not: config/paths (05), error rendering polish (06)

## Detailed Requirements

1. `internal/exitcode`: constants exactly per DESIGN §5.3
   (`OK=0, Generic=1, Usage=2, DockerUnavailable=3, ScenarioInvalid=4, NoActiveRun=5, LocksFailed=6, FloorLocked=7, Internal=10`)
   and `type Coded struct { Code int; Err error }` implementing `error` + `Unwrap`. Helper
   `exitcode.Wrap(code int, err error) error` and `exitcode.From(err error) int` (default `Generic=1`,
   cobra usage errors → 2).
2. Root command `debugdungeon`:
   - Short description: `A debugging escape game: repair broken environments to get out.`
   - `SilenceUsage: true`, `SilenceErrors: true` (main renders errors once).
   - Persistent flags: `--verbose|-v` (bool), `--no-color` (bool), `--data-dir string` (empty default).
     `NO_COLOR` env (any non-empty value) forces no-color (https://no-color.org contract).
3. `main.go`: `err := cli.Execute(ctx)`; render error to stderr (temporary plain rendering until 06),
   `os.Exit(exitcode.From(err))`. Install a top-level `defer` panic trap that prints
   `internal error — please report: <github issues URL>` + stack to the debug log location note, exits 10.
   Wire `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)` so commands get cancellation.
4. `version` command prints one line:
   `debugdungeon <version> (<commit>, <date>, go<goversion>, <GOOS>/<GOARCH>)` from `internal/version`. Also support `--version` on root.
5. `completion` command: cobra defaults for bash/zsh/fish (hidden from main help is fine).
6. Stdout/stderr convention (repo-wide, document in package comment of `internal/cli`): data and
   game output → stdout; progress/status/errors → stderr.

## Acceptance Criteria

- [ ] `debugdungeon --help` exits 0; unknown command/flag exits 2.
- [ ] `debugdungeon version` matches the specified format (regex-tested).
- [ ] A test command returning `exitcode.Wrap(3, err)` makes the process exit 3 (integration-tested via `os/exec` on the built binary or `main`-level test).
- [ ] Induced panic in a hidden test command exits 10 and prints the report message.
- [ ] `NO_COLOR=1` and `--no-color` both set the global color-disable flag (exposed as `cli.ColorDisabled()` for later issues).

## Validation

`go test ./internal/cli/... ./internal/exitcode/...` green; manual transcript of
`--help`, `version`, unknown-command runs attached to PR.

## Dependencies

01.

## Non-goals

Any game command, config file reading (05), styled error rendering (06).

## Design References

DESIGN §5.1–5.3, §4.2; ADR-001, ADR-008.
