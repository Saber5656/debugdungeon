# Title

`play` command: start/resume orchestration and the session loop

## Summary

Implement `debugdungeon play <scenario-id>`: resolve → gate → ensure image → create room → inject
helpers → start → enter the session loop dispatching sentinel outcomes, per DESIGN §3.1–3.3.

## Context

This command composes nearly every engine piece (10, 14, 15, 17, 18, 20, 26) into the core player
experience. All game-loop UX text is born here.

## Scope

- `internal/cli/play.go`, `internal/game/orchestrate.go` + tests
- Not: lock evaluation/victory bookkeeping internals (22), hint/give-up logic (23/25 — invoked via shared funcs)

## Detailed Requirements

1. Argument: exactly one scenario id → `Registry.ByID`; miss → exit 4 with fuzzy suggestion
   ("did you mean rusty-path?" — nearest by Levenshtein ≤ 3, else list hint).
2. Gate: `progress.FloorUnlocked` (26) unless `free_roam`; locked → exit 7 with the unlock rule text.
3. Active-run rules (§3.3): same id → resume (Reconcile; broken → offer reset guidance, exit 1);
   different id → exit 1 with message; `--force` → FIRST validate the target (id resolves, floor
   unlocked) and only then confirm y/N and abandon the current run (stop+remove container, Clear
   run; progress untouched, NO solution reveal) — never abandon before the target is known-startable.
4. Fresh start sequence (each step logged; failure unwinds: remove container if created; state
   mutations wrapped in short `WithLock` sections (20) — the lock is NOT held during the
   interactive session):
   `EnsureImage` (14; while building show spinner line
   `Forging this room for the first time… (docker build, may take a few minutes)`; `built=false` → skip message)
   → Save Run `creating` (container_id empty; 20's sequencing rule) → `CreateRoom` (15) → save
   container_id → `InjectHelpers` (18; motd = banner bytes) → `StartRoom` → save state `running`
   → banner (host prints it exactly once here; re-entries rely on 18's `DD_REENTRY` motd — the
   single-render contract) → loop.
5. Banner (host-side, stdout; same bytes injected as motd by 18): room title, floor/room position,
   difficulty stars, lock names, hints used `n/m`, lore paragraph — **every scenario-sourced field
   passes textsafe (title/lock names SanitizeInline, lore Sanitize; CONVENTIONS C7)** — and the
   instruction line
   `Type escape to attempt escape · hint for a hint · giveup to abandon`
   `(PATH-proof absolutes: /dungeon/bin/escape · /dungeon/bin/hint · /dungeon/bin/giveup)`.
6. Session loop:
   ```
   for {
     outcome := session.Run(...)
     switch outcome.Kind {
     case Escape:  reports := game.Evaluate (22)
                   if allOpen → game.Victory (22); return 0
                   else print closed locks (name + msg, red ✗) and re-enter
     case Hint:    text := game.RevealNextHint (23); print; re-enter
     case Giveup:  confirmed := prompt y/N
                   if confirmed → game.GiveUp (25, reveal); return 0
                   else re-enter
     case Broken:  print room-died message + `debugdungeon reset` guidance; return exit 1
     case Exit:    print pause message ("You step back from the dungeon. The room remains.
                   Resume: debugdungeon play <id> · abandon: debugdungeon give-up"); return 0
     }
   }
   ```
7. Resume path: Reconcile → banner (with elapsed + hints so far) → loop. No image/create steps.
8. Non-TTY → exit 2 before any Docker work (17's check, done early).
9. Ctrl-C during build cancels cleanly (14); during session it passes through to the shell (17).

## Acceptance Criteria

- [ ] Unit (mocked engine interfaces): fresh-start step order + unwind-on-failure (create fails → no run.json, no container leak).
- [ ] Unit: dispatch table for all five outcome kinds; `--force` flow clears old run and container.
- [ ] Gating: locked floor → exit 7 + rule text; free_roam bypass covered.
- [ ] Unknown id → exit 4 with suggestion (test with typo fixture).
- [ ] itest: loop mechanics against a fixture room — play, in-shell `escape` dispatches into the Evaluate branch (mocked until 22 merges; finalized as a real full-loop test in 22's itest), `hint` branch prints and re-enters, plain `exit` pauses with the resume message.

## Validation

`make test` + `make itest`; manual played transcript (script(1) capture) attached to PR.

## Dependencies

10, 14, 15, 16, 17, 18, 20, 26. The loop dispatches into `game.Evaluate`/`Victory` (22),
`RevealNextHint` (23), and `GiveUp` (25): those land in the SAME wave right after this issue
(ISSUE_PLAN ordering note); until each merges, `play` wires the corresponding branch to a typed
`errNotImplemented` stub behind the `game` package interface, and this issue's full-loop itest is
finalized by 22. Also exposes `cli.RunPlay(ctx, deps, scenarioID string, opts PlayOpts) error` as
the reusable entry point (consumed by the TUI map, issue 64).

## Non-goals

`check`-from-second-terminal (22), map/list rendering (27/28), ambience art (65).

## Design References

DESIGN §3.1–3.3, §5.1, §5.3, §9.1, §11 F5/F11; ADR-003.
