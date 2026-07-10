# Title

Interactive session core: PTY exec and sentinel exit codes

## Summary

Implement `internal/session`: the raw-mode interactive `docker exec` session into a room and the
sentinel-exit-code protocol (42/43/44) of ADR-003. Returns a structured Outcome; looping/dispatch
is the `play` command's job (21).

## Context

This is the heart of the single-terminal UX (DESIGN §3.2, §7.5). It must be robust to resize,
Ctrl-C passthrough, container death, and non-TTY invocation (F11).

## Scope

- `internal/session/session.go` + protocol unit tests + itest
- Not: helpers injection (18), the re-entry loop and banners (21)

## Detailed Requirements

1. Types:
   ```go
   type OutcomeKind int // KindExit, KindEscape, KindHint, KindGiveup, KindBroken
   type Outcome struct { Kind OutcomeKind; ExitCode int }
   const (ExitEscape = 42; ExitHint = 43; ExitGiveup = 44)
   ```
2. `Run(ctx, api, containerID string, entry scenario.Entry, stdio Stdio) (Outcome, error)` where
   `type Stdio struct { In *os.File; Out, Err io.Writer }` (In must expose an Fd for raw mode; test
   fakes use a PTY pair). Required `dockerx.API` additions (CONVENTIONS C2): `ExecCreate`,
   `ExecAttach` (hijacked conn), `ExecResize`, `ExecInspect`, `ContainerInspect` — SDK-shaped.
   - Precondition: `stdio.In` must be a terminal (`term.IsTerminal`); else return exported
     `ErrNotInteractive` (matched via `errors.Is`) → CLI maps to exit 2 with message
     "play requires an interactive terminal" (F11). No exec is created.
   - `ExecCreate`: `Cmd=["/bin/bash", "-l"]` (entry.shell is bash-only in schema v1 — SV009;
     ignore other values defensively), `User=entry.User`, `WorkingDir=entry.Workdir`,
     `Tty=true`, attach stdin/out/err, `Env=["DEBUGDUNGEON=1","HISTFILE=<home>/.dungeon_history","TERM=<host $TERM or xterm-256color>"]`
     where `<home>` = `/root` when user root else `/home/<user>`.
   - `ExecAttach` with Tty; put local stdin into raw mode (`term.MakeRaw`), ALWAYS restore on every
     return path (defer + restore before printing anything).
   - Pumps & lifecycle: two goroutines — stdin→conn write, conn read→stdout. The read pump ending
     (EOF/error) is the session-end signal; then CLOSE the hijacked conn (which unblocks the stdin
     pump — never wait on it otherwise), restore the terminal, and only then inspect/print.
     Write/read errors after exec exit (EIO class) are ignored. Initial `ExecResize` to current
     size, then resize on `SIGWINCH` (darwin/linux; small platform file).
   - On session end: `ExecInspect` (retry ≤ 3 × 100ms while `Running`) → exit code → map 42/43/44 →
     Outcome kinds; anything else → `KindExit` with the code.
   - Container-died detection: attach/read error + `ContainerInspect` shows not running →
     `KindBroken` (F5), no error.
   - ctx cancellation (SIGTERM to CLI): close attach, restore terminal, return
     `Outcome{Kind: KindExit, ExitCode: -1}, nil`.
3. Protocol notes in package doc: exit-code channel is intentionally the ONLY inbound signal (TB2);
   codes can collide with player programs (KU-7 — harmless).
4. Testability: exec/attach behind the `dockerx.API` interface; protocol mapping unit-tested with
   a fake returning canned exit codes; raw-mode paths guarded so unit tests run without a TTY.

## Acceptance Criteria

- [ ] Unit: mapping table 0/1/42/43/44/137 → Kind{Exit,Exit,Escape,Hint,Giveup,Exit}; broken-container path → KindBroken.
- [ ] Non-TTY stdin → typed error, no exec created.
- [ ] itest (real daemon, PTY via `creack/pty` test-only dep): open a real bash session, write `exit 43\n` into the PTY → KindHint; echo round-trip works; resize call doesn't error; write `exit\n` → KindExit(0).
- [ ] Terminal state restored after every outcome (itest asserts via `stty -a` compare where feasible; at minimum code-review checklist + deferred restore in all paths).

## Validation

`make test` + `make itest` (the PTY itest above IS the executable smoke; no ad-hoc harness needed).

## Dependencies

13, 15.

## Non-goals

Re-entry loop, status line, banner printing (21), Windows ConPTY.

## Design References

DESIGN §3.2, §3.3, §7.5, §11 F5/F11; ADR-003; KU-7.
