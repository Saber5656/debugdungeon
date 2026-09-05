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
   `exitcode.Wrap(code int, err error) error` and `exitcode.From(err error) int` (default `Generic=1`).
   Usage-error detection: register `root.SetFlagErrorFunc` and argument validators so that flag
   parse errors and unknown commands/args are wrapped as `exitcode.Wrap(2, err)` before reaching
   `From` — runtime errors from `RunE` bodies are never code 2. `internal/exitcode` is a deliberate
   subpackage outside DESIGN §4.2's map (utility, no domain logic — note this in its doc.go).
2. Root command `debugdungeon`:
   - Short description: `A debugging escape game: repair broken environments to get out.`
   - `SilenceUsage: true`, `SilenceErrors: true` (main renders errors once).
   - Persistent flags: `--verbose|-v` (bool), `--no-color` (bool), `--data-dir string` (empty default).
     `NO_COLOR` env (any non-empty value) forces no-color (https://no-color.org contract).
3. `main.go`: `err := cli.Execute(ctx)`; render error to stderr (temporary plain rendering until 06),
   `os.Exit(exitcode.From(err))`. Install a top-level `defer` panic trap that prints
   `internal error — please report: https://github.com/Saber5656/debugdungeon/issues` plus the
   stack trace to stderr (log-file integration arrives with 06), exits 10.
   Wire `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)` so commands get cancellation.
   This issue adds `spf13/cobra` to go.mod/go.sum (first external dependency — ADR-001 budget).
4. `version` command prints one line:
   `debugdungeon <version> (<commit>, <date>, go<goversion>, <GOOS>/<GOARCH>)` from `internal/version`
   (no Docker API version here — that is `doctor`'s job, DESIGN §5.1). Also support `--version` on root.
5. `completion` command: cobra defaults for bash/zsh/fish (hidden from main help is fine).
6. Stdout/stderr convention (repo-wide, document in package comment of `internal/cli`): data and
   game output → stdout; progress/status/errors → stderr.
7. Global-flag access: one accessor struct `cli.Options{Verbose, NoColor bool; DataDir string}`
   retrievable via `cli.Opts(ctx)`; `ColorDisabled()` is sugar over it. No other global state.
8. Test-hook mechanism (used by AC 3/4 and later binary-level tests): hidden commands compiled only
   under build tag `testhooks` (`internal/cli/testhooks.go` with `//go:build testhooks`), providing
   `__exit <code>` (returns `exitcode.Wrap(code, …)`) and `__panic`. Makefile target `build-test`
   builds with `-tags testhooks`; release builds never include the tag (asserted in 36's snapshot smoke).

## Acceptance Criteria

- [ ] `debugdungeon --help` exits 0; unknown command/flag exits 2.
- [ ] `debugdungeon version` matches the specified format (regex-tested).
- [ ] `__exit 3` via the `testhooks` binary makes the process exit 3 (integration-tested via `os/exec` on `make build-test` output).
- [ ] `__panic` exits 10 and prints the report message + stack.
- [ ] `NO_COLOR=1` and `--no-color` both set the global color-disable flag (exposed as `cli.ColorDisabled()` for later issues).
- [ ] A plain `make build` binary does not expose `__exit`/`__panic`.

## Validation

`go test ./internal/cli/... ./internal/exitcode/...` green; manual transcript of
`--help`, `version`, unknown-command runs attached to PR.

## Dependencies

01.

## Non-goals

Any game command, config file reading (05), styled error rendering (06).

## Design References

DESIGN §5.1–5.3, §4.2; ADR-001, ADR-008.
