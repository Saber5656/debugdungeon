# Title

Lock runner: host-driven checks via stdin-piped exec

## Summary

Implement `internal/locks`: sequential execution of a scenario's lock scripts inside the room via
`docker exec` with stdin-piped script bodies, timeouts, capped output capture, and sanitized
player-facing messages.

## Context

Locks are the win condition (DESIGN §6.3, §7.7). Scripts never live in the image (spoiler hygiene
+ integrity); they stream from the scenario FS at check time.

## Scope

- `internal/locks/runner.go` + tests
- Not: victory bookkeeping (22), solvability harness (29 — reuses this runner)

## Detailed Requirements

1. Types:
   ```go
   type Report struct { ID, Name string; Open bool; Msg string; DurationMS int64 }
   ```
2. `RunAll(ctx, api, containerID string, l *scenario.Loaded) ([]Report, error)` (`scenario.Loaded`
   from issue 10; API methods per CONVENTIONS C2 — `ExecCreate`/`ExecAttach`/`ExecInspect`):
   - For each lock in manifest order (sequential):
     - Read script bytes from `l.FS` at `path.Clean(lock.Script)` — the manifest value is ALREADY
       the `checks/…`-relative path (no extra prefixing); defensively reject if it doesn't clean
       to a path under `checks/` (validated by 08 SV013, belt-and-braces here).
     - `ExecCreate`: `Cmd=["timeout", strconv.Itoa(timeoutSec), "/bin/sh", "-s"]`, `User="root"`,
       `Tty=false`, attach stdin+stdout+stderr. Coreutils `timeout` runs INSIDE the container and
       reliably kills the script (Docker's API cannot kill an exec); cookbook §13 guarantees its
       presence. `timeout` exiting 124 → CLOSED with `Msg="lock check timed out after Ns"`.
     - Attach; write script; `CloseWrite`; read demuxed output via `stdcopy` in a bounded goroutine,
       keeping first 4 KiB per stream (excess discarded, truncation noted in the debug log only).
     - Host-side backstop: `context.WithTimeout(ctx, timeout_sec + 2s)`:
       - normal completion → `ExecInspect` exit code: 0 → Open; 124 → timeout-CLOSED as above.
       - backstop trip (in-container timeout gone/wedged) → CLOSE the hijacked conn (unblocks the
         reader), Report `Open=false`, same timeout Msg, log a stray-process warning (PidsLimit
         bounds damage; `reset`/teardown clears strays). Continue with next lock.
   - `Msg` extraction (closed locks): last `MSG: `-prefixed line within the captured (≤4 KiB)
     stdout → suffix; else generic "the lock holds fast". (A MSG line beyond the cap is lost —
     cookbook §6 caps check output anyway.) Always `textsafe.SanitizeInline` (≤ 200 runes).
     `Report.Name` is likewise `SanitizeInline(name, 60)` at construction — every Report field is
     display-safe by construction.
   - Full stdout/stderr (capped) + exit code + duration → debug log for every lock.
   - Error semantics: infrastructure errors (attach failure, container not running) abort with
     error (caller reconciles F5); a failing *script* is a closed lock, not an error.
3. `AllOpen([]Report) bool` helper.
4. Fixture-based itests use a busybox-ish fixture room (template) with inline scripts: pass, fail
   with MSG, fail without MSG, sleep-based timeout, hostile ANSI in MSG (sanitized out).

## Acceptance Criteria

- [ ] itest matrix above passes; timeout case completes in ~timeout (±2s), not script duration, and the timed-out child is actually dead inside the container (the fixture script sleeps under a UNIQUE marker argv, e.g. `sleep 300 # dd-lock-fixture-<random>`; post-timeout exec asserts `pgrep -f <marker>` is empty).
- [ ] Hostile `MSG:` bytes arrive sanitized (no ESC in Report.Msg) — asserted.
- [ ] Output capping proven with a 1 MiB spam script (memory bounded, report notes truncation).
- [ ] Unit tests for MSG parsing and AllOpen with mocked exec.
- [ ] Reports preserve manifest order.

## Validation

`make test` + `make itest` green.

## Dependencies

10 (Loaded), 11, 13, 15.

## Non-goals

Parallel lock execution, retry policies (retry loops belong inside check scripts per cookbook), exec kill workarounds.

## Design References

DESIGN §6.3, §7.7, §10.5; ADR-006 (harness reuse).
