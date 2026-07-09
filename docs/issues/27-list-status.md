# Title

`list` and `status` commands

## Summary

Implement the scenario overview table (`list`) and the active-run summary (`status`), the two
primary read-only surfaces (DESIGN §5.1).

## Context

`list` is the player's menu and must communicate gating clearly; `status` is the second-terminal
companion during play. Both consume registry + progress + run stores read-only.

## Scope

- `internal/cli/list.go`, `internal/cli/status.go`, first pieces of `internal/ui/table.go` + golden tests
- Not: map art (28), pack origin column (61 extends)

## Detailed Requirements

1. `list` output (stdout), grouped by floor in order:
   - Floor header: `Floor 2 — The Daemon Warrens (2/4 cleared)`; locked floor header appends
     `🔒 <UnlockRequirementText>` and its rooms render dimmed with 🔒 status.
   - Row columns: status glyph (`⬜` new / `✅` cleared / `🏳` given_up / `🔒` locked), id,
     title (SanitizeInline, width-capped), ★difficulty, topics (comma), `~NNm` estimate,
     best time `mm:ss` when cleared.
   - Footer: `Active run: <id> (mm:ss elapsed) — resume with: debugdungeon play <id>` when run exists.
   - `--no-color`/`NO_COLOR`: glyphs become ASCII (`[ ]`,`[x]`,`[g]`,`[L]`) — ui helper owns the mapping.
2. `status` output:
   - No active run → print `No active run. Start one: debugdungeon list` → exit 5 (contract §5.3).
   - Else: scenario title/id, floor, state (running/broken), elapsed `mm:ss`, hints `n/m`,
     resets, container short-id, and last lock report table (from run.last_lock_report; `(not yet
     checked)` when empty). Broken state appends reset guidance.
   - `status` performs NO Docker calls (pure state read; staleness acceptable — document; reconcile
     happens in play/check).
3. Both commands work with Docker down (no dockerx import).
4. `internal/ui`: minimal column-width table helper (padding, truncation via SanitizeInline,
   glyph/ASCII mapping, lipgloss styles behind the color flag). Keep API small: `ui.Table(w, rows)`.

## Acceptance Criteria

- [ ] Golden tests (fixture registry with 3 floors + progress states): color and no-color, locked
      and unlocked floors, active-run footer present/absent.
- [ ] `status` golden: running with lock report, broken, no-run (exit 5).
- [ ] Neither command imports `internal/dockerx` (arch test via depguard lint rule or a small `go list` test).
- [ ] Hostile scenario titles render sanitized/truncated without breaking columns.

## Validation

`go test ./internal/cli/... ./internal/ui/...` green; transcripts (color + NO_COLOR=1) in PR.

## Dependencies

10, 20, 26.

## Non-goals

Interactive filtering, JSON output (`--json` deferred v2), pack origin tags (61).

## Design References

DESIGN §3.5, §5.1, §5.3 (exit 5), §10.5.
