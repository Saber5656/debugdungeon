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

1. `GiveUp(ctx, deps, run, reveal bool) error`:
   - progress: `ApplyGiveUp(scenarioID)` (26) — sets `given_up` **unless** already `cleared` (§9.7).
   - Stop+remove container (tolerate gone), `Clear` run, release lock.
   - Caller prints solution afterwards when `reveal`.
2. `give-up` CLI: requires active run (exit 5); y/N confirm
   (`Abandon this room? The solution will be revealed. (--no-reveal to skip)`); flags `--no-reveal`, `--yes`.
   In-session giveup (21) reuses GiveUp with reveal=true after its own confirm.
3. `solution <scenario-id>` CLI:
   - id resolve → exit 4 unknown.
   - progress status must be `cleared` or `given_up`; else exit 1 with
     `Solutions unlock after you clear or give up on a room.` (No floor-gating check — owning the
     status implies access.)
   - Render `solution.md` from scenario FS: sanitized (11), markdown-lite (headings bold, code
     spans styled, fenced blocks indented verbatim-after-sanitize), pager-less (plain stdout).
4. Both flows must work when Docker is down for `solution` (no Docker calls) and degrade for
   `give-up` (container removal failure → warn + still clear run/progress; leftovers to `clean`).
5. Print epilogue after reveal: `The dungeon remembers. Replay anytime: debugdungeon play <id>`.

## Acceptance Criteria

- [ ] Unit: GiveUp ordering (progress → teardown → clear), cleared-not-downgraded rule, teardown-failure still clears run.
- [ ] CLI: solution gate matrix — new(⛔ exit 1), cleared(✅), given_up(✅), unknown id(exit 4).
- [ ] `--no-reveal` skips solution output; `--yes` skips prompt; non-TTY without `--yes` aborts safely (exit 1).
- [ ] Hostile solution.md corpus renders sanitized.
- [ ] itest: full give-up on `_template` room removes container, progress shows given_up, `solution` then prints.

## Validation

`make test` + `make itest` green; transcript of give-up→solution→replay in PR.

## Dependencies

10, 11, 20, 26.

## Non-goals

Partial reveals, per-lock solutions, spoiler encryption.

## Design References

DESIGN §3.4, §5.1, §9.6–9.8, §10.5.
