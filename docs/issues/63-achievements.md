# Title

Achievements engine and initial achievement set

## Summary

Implement a static-predicate achievements engine evaluated at run-terminal time, persisted in
progress.json, surfaced on victory banners and in `stats`.

## Context

Light gamification pull for P1/P2 personas (DESIGN §2.2, wave 9) without any server or clock
gimmicks. Predicates read progress + history + the just-finished run only — deterministic and
testable.

## Scope

- `internal/achievements/achievements.go` + tests; hooks in victory flow (22) and `stats` (62)
- Not: UI art beyond a banner line (65), notifications

## Detailed Requirements

1. Model: `type Achievement struct { ID, Name, Desc string; Pred func(ev Event) bool }` where
   `Event{Run RunRecord; Progress *progress.P; History []Rec; Registry *scenario.Registry}` is
   built ONLY at cleared-terminal (give-up never unlocks). IDs frozen, kebab-case.
2. Initial set (exact IDs):
   - `first-blood` — first cleared room ever.
   - `floor-1-cleared` … `floor-5-cleared` — all rooms of floor N cleared.
   - `throne-taken` — final-cascade cleared.
   - `dungeon-master` — all 20 bundled rooms cleared.
   - `no-torch-needed` — any clear with 0 hints.
   - `untouched` — clear with 0 hints AND 0 resets on difficulty ≥ 3.
   - `speedrunner` — clear in < 50% of time_estimate_min.
   - `necromancer-no-more` — clear zombie-horde without the gravekeeper lock ever failing during
     the run (needs last_lock_report history? NOT AVAILABLE per-attempt → REDESIGN: drop this one;
     replace with:) `comeback` — clear a room previously given up.
   - `pack-rat` — clear a room from an installed pack.
   (11 achievements total.)
3. Storage: `progress.json` `achievements: {"<id>": "<RFC3339 unlocked_at>"}` (26 reserved the
   field). Unlock is idempotent; never removed even if predicates would later fail (e.g. pack uninstalled).
4. Evaluation: `EvaluateOnClear(ev) []Unlocked` — pure; caller (22) persists + returns list to the
   banner renderer: `★ Achievement unlocked: No Torch Needed` (one line each, after the victory banner).
5. `stats` (62) gains an achievements block: unlocked n/11 + names+dates; locked ones show name
   only (descriptions hidden until unlocked — small mystery).
6. Predicates must not read the filesystem/Docker — everything via Event (tested with fixtures).

## Acceptance Criteria

- [ ] Table-driven predicate tests: each achievement has ≥1 unlocking and ≥1 non-unlocking fixture (incl. boundary: exactly 50% time = NOT speedrunner).
- [ ] Idempotence + persistence round-trip (unlock survives reload; re-clear doesn't duplicate).
- [ ] Victory banner renders multi-unlock correctly (clear that triggers 3 at once — fixture).
- [ ] give-up never evaluates (unit).
- [ ] stats block golden (mixed unlocked/locked).

## Validation

`go test ./internal/achievements/...` + itest: tutorial clear unlocks `first-blood` end-to-end.

## Dependencies

26, 62 (22 for the hook).

## Non-goals

Server sync, badges/images, per-achievement rewards, retroactive grants beyond what Event data supports.

## Design References

DESIGN §2.2 (wave 9 gamification), §3.4, §8.2; ADR-007.
