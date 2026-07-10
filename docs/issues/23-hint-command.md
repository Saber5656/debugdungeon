# Title

`hint` command: progressive hint reveal

## Summary

Implement progressive hints: shared `RevealNextHint` used by the in-session sentinel (21) and the
standalone `debugdungeon hint` command.

## Context

DESIGN §3.4/§9.9: hints reveal strictly in order, are persisted on the run, counted into the clear
record, and rendered sanitized.

## Scope

- `internal/game/hints.go`, `internal/cli/hint.go` + tests
- Not: hint authoring rules (12), penalties/scoring (none in v1)

## Detailed Requirements

1. `RevealNextHint(deps, run, l *scenario.Loaded) (HintOut, error)`:
   - `idx := run.HintsRevealed`; if `idx >= len(hints)` → `HintOut{Exhausted: true}` (no state change).
   - Else read `hints/<file>` from `l.FS`, `run.HintsRevealed++`, save run (atomic, 20).
   - `HintOut{Index (1-based), Total, Body string}` — body sanitized (11) then rendered
     markdown-lite: `**bold**` and backtick spans styled via ui when color on; plain otherwise.
2. Re-reveal behavior: `hint` never re-prints earlier hints; `debugdungeon hint --all` prints all
   *already revealed* hints (no state change) — for players returning after a pause.
3. `hint` command: no active run → exit 5; wraps the reveal-increment save in `WithLock` (20,
   short section — works from a second terminal during play); renders
   `Hint 2/3:` header + body indented two spaces.
4. In-session path (21) calls the same function; output written above the re-entered shell.
5. Exhausted rendering: `No more hints. The dungeon expects you to prevail — or type giveup.`

## Acceptance Criteria

- [ ] Unit: reveal sequence 0→1→2, persistence after each, exhausted no-op, `--all` after two reveals prints both without increment.
- [ ] Hostile hint file (ANSI/OSC bytes) renders sanitized (corpus reuse from 11).
- [ ] No run → exit 5; run in `broken` state → hints still allowed (player may want a nudge before reset) — covered.
- [ ] Golden: header + indent formatting, color and no-color.

## Validation

`go test ./internal/game/... ./internal/cli/...` green; manual transcript showing 3-hint scenario
flow in PR.

## Dependencies

10, 11, 20.

## Non-goals

Hint penalties, timed hints, per-lock hints.

## Design References

DESIGN §3.4, §5.1, §9.9, §10.5.
