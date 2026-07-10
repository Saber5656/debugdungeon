# Title

`give-up` and `solution` commands

## Summary

Implement abandoning a run with optional solution reveal, and the standalone `solution` command
gated on cleared/given-up status (DESIGN §9.6–9.8).

## Context

Give-up is the pressure-release valve that keeps stuck players in the game; the solution gate
protects spoilers just enough (anti-cheat is a non-goal).

## Scope

- `internal/game/giveup.go`, `internal/cli/giveup.go`, `internal/cli/solution.go` + tests
- Not: solution.md authoring (content issues), rendering engine beyond markdown-lite (23's helper reused)

## Detailed Requirements

1. `GiveUp(ctx, deps, run, reveal bool) error` (valid from `running` and `broken` via `CanRun`;
   acquires its own short `WithLock` sections — callers do NOT wrap it):
   - progress: `ApplyGiveUp(scenarioID, now)` (26) — sets `given_up` **unless** already `cleared` (§9.7).
   - Stop+remove container (tolerate gone AND daemon-down: teardown errors → warn, leftovers to
     `clean`), `Clear` run.
   - Caller prints solution afterwards when `reveal`.
   - This issue also replaces 21's giveup-branch stub (small `play.go` wiring diff in scope).
2. `give-up` CLI: requires active run (exit 5); y/N confirm
   (`Abandon this room? The solution will be revealed. (--no-reveal to skip)`); flags `--no-reveal`, `--yes`.
   In-session giveup (21) reuses GiveUp with reveal=true after its own confirm.
3. `solution <scenario-id>` CLI:
   - id resolve → exit 4 unknown.
   - progress status must be `cleared` or `given_up`; else exit 1 with
     `Solutions unlock after you clear or give up on a room.` (No floor-gating check — owning the
     status implies access.)
   - Render `solution.md` from scenario FS with a small dedicated renderer defined HERE (23's
     hint renderer covers only bold/code spans): `Sanitize(raw, 16384)` first, then line-based
     markdown-lite — `#`-headings → bold line, backtick spans styled, fenced blocks emitted
     verbatim-post-sanitize indented 4 spaces, everything else literal; color off → plain.
     Golden fixtures for color/plain. Pager-less (plain stdout). Malformed markdown = literal text
     (no parser errors possible by construction).
4. Both flows must work when Docker is down for `solution` (no Docker calls) and degrade for
   `give-up` (container removal failure → warn + still clear run/progress; leftovers to `clean`).
5. Print epilogue after reveal: `The dungeon remembers. Replay anytime: debugdungeon play <id>`.

## Acceptance Criteria

- [ ] Unit: GiveUp ordering (progress → teardown → clear), cleared-not-downgraded rule, teardown-failure still clears run; daemon-unavailable give-up still marks progress + clears run + exits 0 with warning (mocked API error).
- [ ] CLI: solution gate matrix — new(⛔ exit 1), cleared(✅), given_up(✅), unknown id(exit 4).
- [ ] `--no-reveal` skips solution output; `--yes` skips prompt; non-TTY without `--yes` aborts safely (exit 1).
- [ ] Hostile solution.md corpus renders sanitized.
- [ ] itest: full give-up on `_template` room removes container, progress shows given_up, `solution` then prints.

## Validation

`make test` + `make itest` green; transcript of give-up→solution→replay in PR.

## Dependencies

10, 11, 20, 21 (stub replaced), 23 (renderer precedent), 26.

## Non-goals

Partial reveals, per-lock solutions, spoiler encryption.

## Design References

DESIGN §3.4, §5.1, §9.6–9.8, §10.5.
