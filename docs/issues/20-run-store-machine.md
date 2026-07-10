# Title

Run store, run state machine, reconciliation, and single-instance lock

## Summary

Implement `internal/game` run persistence (`run.json`, DESIGN §8.3), the run state machine
(DESIGN §7.3), startup reconciliation against actual Docker state, and the advisory lock that
prevents two CLIs from driving one run.

## Context

Every run-scoped command (21–25) mutates state through this store. Crash-safety (F5/F7/F8/F9) is
decided here.

## Scope

- `internal/game/run.go`, `internal/game/store.go`, `internal/game/reconcile.go` + tests
- Replaces the internals of `game.ActiveContainerID` (introduced in 16) with the real store, same signature
- Not: any CLI command wiring (21–25), progress (26)

## Detailed Requirements

1. `type Run struct` — fields exactly per DESIGN §8.3 (schema_version, run_id, scenario_id, source,
   content_hash, image_ref, container_id, state, started_at, hints_revealed, resets,
   last_lock_report). `run_id`: `r-` + 6 hex random.
2. States & transitions (table-driven; illegal transition = programming error returned as typed err):
   `creating → running`; `running → checking → running|escaped`; `running → resetting → running`;
   `running → given_up`; `running|checking|resetting → broken`; `broken → resetting`;
   terminal: `escaped`, `given_up` (store cleared, not persisted as states).
   "Paused" is NOT a state: a detached shell leaves the run `running` (DESIGN §7.3). Implement the
   §7.3 command-eligibility table as `CanRun(cmd, state) error` (single source for 21–25's guards;
   transient states → typed `ErrBusy` with the retry message).
3. Store:
   - `Load(paths) (*Run, error)` — missing file → `(nil, nil)`; corrupt → quarantine to
     `run.json.corrupt-<unix-ts>`, return `(nil, WarnCorrupt)` sentinel the CLI prints once (§8.4).
   - `Save(paths, *Run)` — atomic: temp file (0600) in same dir → fsync → rename; dir perms 0700 (05).
   - `Clear(paths)` — remove run.json (idempotent).
4. Advisory lock (F9, short-scoped): `WithLock(paths, func() error) error` — `O_CREATE` `run.lock`
   + `unix.Flock(LOCK_EX)` with a 2s acquire deadline (poll LOCK_NB every 100ms); deadline →
   typed `ErrBusy` ("another debugdungeon command is mid-operation — retry in a moment").
   Held ONLY around load-mutate-save critical sections and engine mutations (evaluate, reset,
   give-up bookkeeping) — NEVER across the interactive shell attach, so second-terminal
   `check`/`hint` work during play (DESIGN §3.3, §11 F9). Read-only `status`/`list` skip it.
5. `Reconcile(ctx, api, run) (*Run, error)`:
   - `ContainerInspect(run.ContainerID)`: not found → state `broken`, container_id kept for message.
   - found & running → unchanged.
   - found & exited → one `ContainerStart` attempt; success → running; failure → `broken`.
   - `broken` is recoverable only via reset (24) or give-up (25); message text owned by callers.
6. `ActiveContainerID` now reads through Load (nil-safe).
7. Elapsed helper: `run.Elapsed(now) time.Duration`.

## Acceptance Criteria

- [ ] Transition matrix unit-tested: every legal edge passes; ≥ 5 illegal edges rejected.
- [ ] Corrupt-file test: garbage bytes → quarantined file exists, Load returns nil + warning sentinel.
- [ ] Atomicity: crash-simulation test (write temp, no rename) leaves previous run.json intact.
- [ ] flock tests: WithLock serializes two goroutines (new fds); a holder past 2s makes the second caller ErrBusy; lock is NOT held during a simulated long session (concurrent WithLock succeeds while "session" runs).
- [ ] Reconcile paths (missing/running/exited-restartable/exited-dead) covered with mocked API.
- [ ] File modes: run.json 0600.

## Validation

`go test ./internal/game/...` green (no Docker needed — mocked API); flock behavior additionally
smoke-tested via two real processes in itest.

## Dependencies

05, 06, 13.

## Non-goals

Multiple concurrent runs (v1: exactly one), run history (62), progress merge (26).

## Design References

DESIGN §7.3, §8.3, §8.4, §9.1–9.3, §11 F5/F7/F8/F9.
