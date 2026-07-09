# Title

`map` command: static dungeon map rendering

## Summary

Render the dungeon as a vertical ASCII/Unicode map — floors as strata, rooms as cells with state
glyphs — including gating status and overall completion (DESIGN §3.5, §5.1).

## Context

The map is the game-feel anchor ("descending a dungeon") that `list` can't convey. v1 is a static
render; Wave 9 (64) upgrades it to an interactive TUI reusing this layout model.

## Scope

- `internal/cli/map.go`, `internal/ui/map.go` + golden tests
- Not: interactivity (64), per-floor art (65)

## Detailed Requirements

1. Layout (box-drawing chars; ASCII fallback under no-color/`--ascii`):
   ```
   ═══ DebugDungeon ═══  7/20 rooms · Floor 3 reached
   ┌ Floor 1 — The Entrance Halls ──────── 4/4 ✦
   │  [✅ welcome-cell] [✅ rusty-path] [✅ forbidden-scroll] [✅ broken-symlink]
   ├ Floor 2 — The Daemon Warrens ──────── 3/4
   │  [✅ …] [⬜ port-poltergeist] …
   ├ Floor 3 — The Flooded Archives ────── 0/3  ← you are here
   │  …
   ├ Floor 4 — 🔒 Clear 2 rooms on Floor 3 to descend.
   ├ Floor 5 — 🔒 …
   └ ★ The Cascade Throne — 🔒 Clear 12 rooms in total…
   ```
   - Locked floors: header only (rooms hidden — mystery preserved).
   - `← you are here`: deepest unlocked floor; active run marks its room with `▶`.
   - Completion `✦` on fully-cleared floors.
2. Data model: `ui.BuildMap(reg, prog, run, colors bool) string` pure function → golden-testable.
3. Wrap room cells to terminal width (from `term.GetSize`; default 80 when unavailable); minimum
   supported width 60 (below → simple list fallback = `list` hint message).
4. Room ids in cells; titles appear in `list` (map stays scannable).
5. Sanitization: ids are regex-safe by schema; titles not rendered here — no untrusted text besides
   none (comment this).

## Acceptance Criteria

- [ ] Golden renders: fresh player (only F1 visible unlocked), mid-game (mixed), all-clear (capstone visible), active-run marker, ASCII fallback, width-40 fallback message.
- [ ] `map` exits 0 with no Docker running (no dockerx import — same arch test as 27).
- [ ] Unlock requirement strings come from 26's `UnlockRequirementText` (no duplicated literals — grep test).

## Validation

`go test ./internal/ui/... -run Map` green; screenshots (color + ASCII) in PR.

## Dependencies

10, 26.

## Non-goals

Scrolling, mouse, animation (64), lore excerpts on map.

## Design References

DESIGN §3.5, §5.1; ISSUE_PLAN wave 9 (64 builds on this).
