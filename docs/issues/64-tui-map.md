# Title

Interactive TUI dungeon map (bubbletea)

## Summary

Upgrade `map` to an interactive bubbletea TUI when stdout is a TTY: navigate rooms, view details,
and launch a room — falling back to the static render (28) otherwise.

## Context

Wave 9 polish for the dungeon feel (DESIGN §5.1 map row: "3 static / 9 TUI"). First and only
bubbletea dependency entry point (ADR-001 deferred it here; KU-10 API churn watched).

## Scope

- `internal/ui/tuimap/` (bubbletea model), wiring in `internal/cli/map.go`
- Adds deps: `charmbracelet/bubbletea` (+ transitives) — record in ADR-001 budget note
- Not: any other command going TUI, mouse support

## Detailed Requirements

1. Activation: `map` with TTY stdout and `!ColorDisabled` → TUI; else static (28). Flag `--static` forces old behavior.
2. Layout: left pane = floors/rooms tree (same data model as 28's BuildMap inputs); right pane =
   detail of selection (title, lore first ~200 chars sanitized, ★, topics, est, status, best
   time/hints/resets, lock names with last-known states if the active run is this room).
3. Keys: `↑/↓/j/k` move, `←/→/h/l` collapse/expand floor, `enter` = action on room
   (unlocked+no-active-run → quit TUI and print `▶ starting…` then EXEC the play flow in-process;
   locked → flash the unlock rule; active-run-elsewhere → flash message), `q`/`esc` quit, `?` keymap overlay.
4. Locked floors: visible headers, rooms hidden (28 parity).
5. Resize handled (WindowSizeMsg); min 60×15 → below that, quit-with-static-fallback.
6. In-process play handoff: after `tea.Quit` completes and terminal is restored, call the play
   command's entry function directly (no self-exec) — run-lock and TTY state must be clean
   (integration-test the handoff: TUI → play → immediate `exit` → back to shell sane).
7. Testing: model Update/View unit tests via bubbletea's test-friendly Model interface (send key
   msgs, golden the View string with fixed size + NO_COLOR-style plain styles); the play handoff
   via itest PTY (34's helpers reused).

## Acceptance Criteria

- [ ] Golden View states: fresh player, mid-game, all-clear, locked-floor flash, keymap overlay (fixed 80×24).
- [ ] Enter-on-unlocked launches play and lands in a working session (PTY itest, template room); terminal restored after exiting.
- [ ] Non-TTY / `--static` / NO_COLOR → identical bytes to 28's output (regression goldens kept).
- [ ] Resize storm (10 rapid WindowSizeMsg) doesn't panic (unit).
- [ ] Dependency budget note updated in ADR-001 (comment PR link).

## Validation

`make test` + PTY itest; short screen recording (or vhs) attached to PR.

## Dependencies

26, 28 (21 for the handoff; 34's PTY test helpers).

## Non-goals

Mouse, animations, ambience art (65), replacing list/status with TUIs.

## Design References

DESIGN §5.1, §3.5; ADR-001 (dep budget); KU-10.
