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
   detail of selection — EVERY scenario-sourced string sanitized per C7: title (SanitizeInline 60),
   lore excerpt (Sanitize, first 200 runes), topics, lock names (SanitizeInline 60 — already
   display-safe when coming from 19's reports), ★, est, status, best time/hints/resets.
   Deterministic model contract (golden-testable): initial selection = first unlocked not-cleared
   room (fallback: first room); floors ascend; rooms in registry order; nothing collapsed
   initially; selection skips hidden (locked-floor) rooms; flash messages last until the next
   keypress; `?` overlay dismissed by any key.
3. Keys: `↑/↓/j/k` move, `←/→/h/l` collapse/expand floor, `enter` = action on room
   (unlocked+no-active-run → quit TUI and print `▶ starting…` then call `cli.RunPlay(ctx, deps,
   id, opts)` — the named entry point 21 exposes; no self-exec, no cobra re-dispatch;
   locked → flash the unlock rule; active-run-elsewhere → flash message), `q`/`esc` quit, `?` keymap overlay.
4. Locked floors: visible headers, rooms hidden (28 parity).
5. Resize handled (WindowSizeMsg); if the window is < 60×15 at startup, don't enter the TUI at all —
   render 28's static map + a "widen your terminal for the interactive map" note, exit 0. A shrink
   below threshold DURING the TUI shows a "terminal too small" holding screen (no crash) and
   restores when widened.
6. In-process play handoff: after `tea.Quit` returns and the terminal is restored, call
   `cli.RunPlay` (21) directly. Sanitize titles/lore before display (see §3 security). Integration-
   test the handoff: TUI → enter → play → immediate `exit` → shell restored and sane.
7. Security: sanitize EVERY scenario-derived string the TUI renders — title, topics, lock names,
   lore excerpt, and any flash text built from them — via `textsafe`/SanitizeInline with caps
   (§10.5 / CONVENTIONS C7); ids are schema-safe.
7. Testing: model Update/View unit tests via bubbletea's test-friendly Model interface (send key
   msgs, golden the View string with fixed size + NO_COLOR-style plain styles); the play handoff
   via itest PTY (34's helpers reused).

## Acceptance Criteria

- [ ] Golden View states: fresh player, mid-game, all-clear, locked-floor flash, keymap overlay (fixed 80×24).
- [ ] Enter-on-unlocked launches play and lands in a working session (PTY itest, template room); terminal restored after exiting.
- [ ] Non-TTY / `--static` / NO_COLOR / width<60 → calls 28's `BuildMap` with the same
  `MapOpts` the `map` CLI would pass (width from `term.GetSize`, ASCII=color-disabled) → identical
  bytes to 28's output (regression goldens kept).
- [ ] Resize storm (10 rapid WindowSizeMsg) doesn't panic (unit).
- [ ] Dependency budget note updated in ADR-001 (comment PR link).

## Validation

`make test` + PTY itest; short screen recording (or vhs) attached to PR.

## Dependencies

21 (`cli.RunPlay` handoff), 26, 28, 34 (PTY test helpers).

## Non-goals

Mouse, animations, ambience art (65), replacing list/status with TUIs.

## Design References

DESIGN §3.3, §3.5, §5.1, §7.5, §10.5; ADR-001 (dep budget); KU-10.
