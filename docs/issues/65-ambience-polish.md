# Title

In-dungeon ambience polish: banners, per-floor art, prompt theming

## Summary

Give the dungeon its atmosphere: per-floor ASCII banner art, a themed in-room shell prompt,
richer victory/give-up screens, and consistent styling — all within NO_COLOR/plain fallbacks.

## Context

Wave 9 game-feel work (DESIGN §2.2). Strictly presentational: no game-rule changes, no new state.
Touches the helper-injection snippet (18) and victory rendering (22).

## Scope

- `internal/ui/art.go` (embedded art data), updates to banner composition (21), victory/give-up
  renders (22/25), profile.d snippet (18)
- Not: sound, animations, TUI map (64)

## Detailed Requirements

1. **Per-floor banner art**: `ui.ArtForFloor(floor int) []string` returning one engine-owned ASCII
   piece per floor 1–6 (≤ 12 lines × ≤ 60 cols; invalid floor → empty slice). `ui.RenderRoomBanner`
   composes: art (trusted) → title/lore (sanitized, C7) → instruction line (trusted), in that
   fixed order; used by `play` (21) and as the injected motd bytes. Art strings contain NO color
   codes (plain ASCII); color is applied by the lipgloss/theme layer only when enabled.
2. **Prompt theming**: change 18's `InjectHelpers` signature to
   `InjectHelpers(ctx, api, containerID string, opts InjectOpts)` where
   `InjectOpts{Motd []byte; RoomID string; Color bool}` (this issue owns the 18-caller update in
   21). The profile.d snippet appends, guarded for interactive bash only
   (`case $- in *i*)`), exactly:
   - Color: `PS1='\[\e[35m\]⚔ <room-id>\[\e[0m\]:\w\$ '`
   - Plain (Color=false): `PS1='[<room-id>] \w\$ '`
   `<room-id>` substituted at inject time (schema-safe chars only). Placed AFTER the PATH/motd
   lines; MUST NOT alter PATH behavior — rusty-path (31) solvability matrix is re-run as a
   regression gate (AC below).
3. **Victory screen v2** (`RenderVictory` extended, 22): exact fields/order — framed header line,
   then `ESCAPED in mm:ss · N hints[· M resets]`, then best-time line (first/new-best/kept per
   `Cleared.FirstClear`/`BestUpdated`), then each achievement as `★ Achievement unlocked: <Name>`
   (63), then `Next: debugdungeon play <id>` signpost (omitted when empty). Give-up render: somber
   framed header + `The solution is revealed below.` intro. Plain and color glyph/label variants
   specified in the goldens.
4. **Style tokens**: centralize colors/glyphs in `internal/ui/theme.go` (single source;
   list/status/map/victory/giveup all consume it) — refactor existing usages; goldens updated once.
5. Nothing scenario-sourced enters art paths (sanitizer untouched); banner composition order:
   art (trusted) → title/lore (sanitized) → instructions (trusted).
6. Full-flow goldens re-recorded: play banner, victory, give-up, list/map spot-checks (color + plain).

## Acceptance Criteria

- [ ] 6 art pieces render within size bounds at 80 cols; plain-mode goldens byte-stable.
- [ ] PS1 shows in-room (PTY itest: prompt contains room id); rusty-path solvability matrix still green (regression evidence).
- [ ] Victory/give-up goldens (with + without achievements, color + plain).
- [ ] Theme tokens: a Go arch test (not a shell grep) asserts no `lipgloss.Color(`/`lipgloss.AdaptiveColor{` literal appears in `internal/ui/*.go` except `theme.go` (walks the package via `go/ast`, ignores `_test.go`).
- [ ] No new deps; binary size delta < 100 KB.

## Validation

`make test` + scenario matrix on 31 + PTY prompt itest; before/after screenshots in PR.

## Dependencies

10, 18, 21, 22, 25, 27, 28, 63 (achievement lines; all listed packages have UI surfaces this issue refactors onto the theme tokens).

## Non-goals

Scenario-supplied art (stays forbidden — TB3), seasonal themes, emoji-heavy output (glyph budget stays per 27).

## Design References

DESIGN §2.2, §3.2, §10.5 (trusted vs sanitized banner layers); COOKBOOK §13 (/dungeon reserved).
