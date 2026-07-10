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
   `Event{Run progress.RunRecord /* 62's history type, the just-finished cleared run */;
   Clear progress.ClearResult /* 26's — carries PrevStatus for comeback */;
   Progress *progress.Progress /* POST-clear snapshot */;
   History []progress.RunRecord /* INCLUDING the current run's record */;
   Registry *scenario.Registry; Estimate int /* time_estimate_min of the room */}`.
   Victory ordering (22's hook, frozen): `ApplyClear` → `AppendRun` → `EvaluateOnClear` — so
   Progress/History are always post-state, and `Clear.PrevStatus`/`FirstClear` carry the pre-state
   facts predicates need. Built ONLY at cleared-terminal (give-up never unlocks). IDs frozen, kebab-case.
2. Initial set — exactly THIRTEEN achievements, exact IDs and predicates:
   - `first-blood` — this clear made distinct-cleared count == 1 (`Clear.FirstClear` && exactly
     one cleared id in Progress).
   - `floor-1-cleared` … `floor-5-cleared` (5 IDs) — all registry rooms of floor N now cleared.
   - `throne-taken` — final-cascade cleared.
   - `dungeon-master` — all 20 bundled rooms cleared.
   - `no-torch-needed` — this clear used 0 hints.
   - `untouched` — this clear: 0 hints AND 0 resets AND room difficulty ≥ 3.
   - `speedrunner` — `Run.ElapsedSec * 2 < Estimate * 60` (strict; exactly 50% does NOT unlock).
     Estimate is schema-guaranteed 5–120 (SV007) — no missing-value case.
   - `comeback` — `Clear.PrevStatus == "given_up"`.
   - `pack-rat` — `strings.HasPrefix(Run.Source, "pack:")` (no dependency on pack issues — the
     Source string exists from 20; naturally unlockable only once packs exist).
3. Storage: `progress.json` `achievements: {"<id>": "<RFC3339 unlocked_at>"}` (26 reserved the
   field). Unlock is idempotent; never removed even if predicates would later fail (e.g. pack uninstalled).
4. Evaluation: `EvaluateOnClear(ev) []Unlocked` — pure; caller (22) persists + returns list to the
   banner renderer: `★ Achievement unlocked: No Torch Needed` (one line each, after the victory banner).
5. `stats` (62) gains an achievements block: unlocked n/13 + names+dates; locked ones show name
   only (descriptions hidden until unlocked — small mystery).
6. Predicates must not read the filesystem/Docker — everything via Event (tested with fixtures).

## Acceptance Criteria

- [ ] Table-driven predicate tests: each achievement has ≥1 unlocking and ≥1 non-unlocking fixture (incl. boundary: exactly 50% time = NOT speedrunner).
- [ ] Idempotence + persistence round-trip (unlock survives reload; re-clear doesn't duplicate).
- [ ] Victory banner renders multi-unlock correctly (clear that triggers 3 at once — fixture).
- [ ] give-up never evaluates (unit).
- [ ] stats block golden (mixed unlocked/locked).

## Validation

`go test ./internal/achievements/... ./internal/progress/... ./internal/game/... ./internal/cli/...`
(persistence + victory-hook + stats-block integration) + itest: tutorial clear unlocks
`first-blood` end-to-end.

## Dependencies

22 (victory hook surface), 26, 62.

## Non-goals

Server sync, badges/images, per-achievement rewards, retroactive grants beyond what Event data supports.

## Design References

DESIGN §2.2 (wave 9 gamification), §3.4, §8.2; ADR-007.
