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

1. `RevealNextHint(deps Deps, run *game.Run, l *scenario.Loaded) (HintOut, error)` — `Deps`
   carries `config.Paths` + the run store (20); allowed run states: `running`, `broken`
   (20's `CanRun`). Behavior:
   - `idx := run.HintsRevealed`; if `idx >= len(l.Spec.Hints)` → `HintOut{Exhausted: true}` (no state change).
   - Else read exactly `l.Spec.Hints[idx].File` from `l.FS` (the manifest value is already the
     `hints/NN.md` relative path — no prefixing; read cap 4 KiB per SV015), under `WithLock`:
     `run.HintsRevealed++`, save (atomic, 20). Read errors → typed error, no state change.
   - `HintOut{Index, Total int; Body string}` (Index 1-based) — Body = `textsafe.Sanitize(raw, 4096)`,
     then rendered markdown-lite per DESIGN §6.5: bold/emphasis + backtick spans styled via ui when
     color on; plain otherwise (§6.5's "bold/emphasis only" plus code spans — DESIGN §6.5 wording
     updated to match in this issue's PR).
2. Re-reveal behavior: `hint` never re-prints earlier hints; `debugdungeon hint --all` prints all
   *already revealed* hints in order, each with its `Hint k/n:` header, no state change (works in
   `broken` too; zero revealed → same friendly line as the no-hints-yet case, exit 0). Exhausted +
   `--all` → prints all hints (all are revealed by definition).
3. `hint` command: no active run → exit 5; wraps the reveal-increment save in `WithLock` (20,
   short section — works from a second terminal during play); renders
   `Hint 2/3:` header + body indented two spaces.
4. In-session path: 21's hint branch already dispatches to this function (this issue replaces
   21's stub — the one-line `play.go` wiring diff is in scope here); output written above the
   re-entered shell.
5. Exhausted rendering: `No more hints. The dungeon expects you to prevail — or type giveup.`

## Acceptance Criteria

- [ ] Unit: reveal sequence 0→1→2, persistence after each, exhausted no-op, `--all` after two reveals prints both without increment.
- [ ] Hostile hint file (ANSI/OSC bytes) renders sanitized (corpus reuse from 11).
- [ ] No run → exit 5; run in `broken` state → hints still allowed (player may want a nudge before reset) — covered.
- [ ] Golden: header + indent formatting, color and no-color.

## Validation

`go test ./internal/game/... ./internal/cli/...` green (3-hint flow covered by a fixture scenario
via `fstest.MapFS` — no bundled-content dependency); manual transcript optional.

## Dependencies

10, 11, 20.

## Non-goals

Hint penalties, timed hints, per-lock hints.

## Design References

DESIGN §3.4, §5.1, §9.9, §10.5.
