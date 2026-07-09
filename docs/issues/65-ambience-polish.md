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

1. **Per-floor banner art**: 6 small ASCII pieces (≤ 12 lines × ≤ 60 cols, engine-owned — NOT
   scenario-supplied; scenarios remain data-only) keyed by floor; rendered above the room banner
   in `play` and in the injected motd. Art must degrade to plain ASCII (it IS ascii) and respect
   `--no-color` (no color codes embedded in art strings; styling applied by lipgloss layer).
2. **Prompt theming**: extend 18's profile.d snippet to set
   `PS1='\[\e[…\]⚔ <room-id>\[\e[0m\]:\w\$ '` for bash when `DEBUGDUNGEON=1` — room id baked at
   injection time; plain variant (`[room-id] \w\$`) when the host had NO_COLOR at inject time.
   MUST NOT break scenarios asserting on PATH/profile (rusty-path 31: verify its locks unaffected — regression run).
3. **Victory screen v2**: floor-themed frame around the existing facts (22's Cleared data),
   achievements lines (63) integrated, next-room suggestion styled as a signpost. Give-up screen
   gets a somber variant + solution intro line.
4. **Style tokens**: centralize colors/glyph choices in `internal/ui/theme.go` (single source;
   list/status/map/victory all consume) — refactor existing usages; snapshot tests updated once.
5. Nothing scenario-sourced enters art paths (sanitizer untouched); banner composition order:
   art (trusted) → title/lore (sanitized) → instructions (trusted).
6. Full-flow goldens re-recorded: play banner, victory, give-up, list/map spot-checks (color + plain).

## Acceptance Criteria

- [ ] 6 art pieces render within size bounds at 80 cols; plain-mode goldens byte-stable.
- [ ] PS1 shows in-room (PTY itest: prompt contains room id); rusty-path solvability matrix still green (regression evidence).
- [ ] Victory/give-up goldens (with + without achievements, color + plain).
- [ ] Theme tokens: grep shows no raw lipgloss color literals outside theme.go (arch test).
- [ ] No new deps; binary size delta < 100 KB.

## Validation

`make test` + scenario matrix on 31 + PTY prompt itest; before/after screenshots in PR.

## Dependencies

10, 18, 22 (63 for achievement lines; 21/25 renders touched).

## Non-goals

Scenario-supplied art (stays forbidden — TB3), seasonal themes, emoji-heavy output (glyph budget stays per 27).

## Design References

DESIGN §2.2, §3.2, §10.5 (trusted vs sanitized banner layers); COOKBOOK §13 (/dungeon reserved).
