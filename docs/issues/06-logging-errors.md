# Title

Debug logging sink and user-facing error presentation

## Summary

Implement `internal/logx`: a JSON debug log file with size-capped rotation, and the standard
user-facing error renderer (one-line summary + hint + optional verbose detail) used by `main`.

## Context

DESIGN §11 requires friendly errors ("next command to try") with stack traces only under
`--verbose`; build/exec transcripts (14, 19) need a durable debug sink for bug reports.

## Scope

- `internal/logx/logx.go`, `internal/logx/render.go` + tests; wire into `main.go` (replacing 04's temporary rendering)
- Not: per-command error texts (owned by each command's issue)

## Detailed Requirements

1. Logger: `logx.Setup(logFile string, verbose bool) (*slog.Logger, func() error)` —
   `slog` JSON handler writing to `LogFile` (from 05), level Debug. When `verbose`, additionally
   mirror level ≥ Info to stderr in plain text. Never write log lines to stdout.
2. Rotation: before opening, if file > 5 MiB, rename to `debug.log.old` (replace existing). No
   external dependency.
3. `type UserError struct { Summary string; Detail string; Hint string; Code int }` implementing
   `error`; integrates with `exitcode.From` (Code wins when set).
4. `logx.Render(w io.Writer, err error, verbose bool)`:
   - line 1: `error: <Summary>` (red unless color disabled)
   - line 2 (if Hint): `try: <Hint>`
   - verbose: append Detail and `%+v` chain.
   - Non-UserError errors render as `error: <err.Error()>` + generic hint pointing to
     `debugdungeon doctor` and the debug log path.
5. Every subsequent issue's commands receive a ready `*slog.Logger` via the cli context; document
   the accessor (`cli.Logger(ctx)`).
6. Sanitization contract: `Render` and the logger are *trusted-text* sinks. Callers must sanitize
   scenario-sourced strings BEFORE passing (cross-reference issue 11); add this rule to package docs.

## Acceptance Criteria

- [ ] Log file created 0600 under the 05 log dir; JSON lines parse; rotation test (write >5 MiB, reopen, `.old` exists).
- [ ] `Render` golden tests: with/without hint, verbose on/off, color on/off.
- [ ] `main` renders `UserError{Summary:"docker daemon unreachable", Hint:"start Docker Desktop, then run: debugdungeon doctor", Code:3}` as two lines and exits 3 (binary-level test).
- [ ] `version` command still writes nothing to the log (lazy open on first write).

## Validation

`go test ./internal/logx/...`; binary transcript demonstrating the two-line error + exit code.

## Dependencies

04, 05.

## Non-goals

Telemetry (never — ADR-007), log shipping, i18n.

## Design References

DESIGN §11 (error UX), §8.1 (log path), §10.5 (sanitize-before-sink rule).
