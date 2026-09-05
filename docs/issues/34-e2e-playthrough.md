# Title

End-to-end PTY playthrough test

## Summary

Add an expect-style E2E test that drives the **real built binary** through a full `welcome-cell`
playthrough over a pseudo-terminal, in CI with a real Docker daemon.

## Context

Unit and harness tests don't cover the assembled UX: raw-mode session, sentinel round-trips,
banners, victory, progress persistence. This is the MVP release smoke (DESIGN §12 E2E row).

## Scope

- `test/e2e/play_test.go` (build tag `e2e`), Makefile target `e2e`, CI job in `.github/workflows/ci.yml` or `scenarios.yml`
- Test-only dependency `github.com/creack/pty` (record in ADR-001's budget note as test-only)

## Detailed Requirements

1. Test flow (timeout 10m total):
   1. Build binary (`make build`) or accept `DD_E2E_BIN` env pointing at one.
   2. `DEBUGDUNGEON_HOME=$(mktemp -d)`.
   3. Spawn `debugdungeon play welcome-cell` under a pty (80×24). Expect (regex, ≤ 120s for the
      first-build step): room banner `Floor 1 — .*welcome-cell|The Welcome Cell` and the instruction line.
   4. Send `hint\r` → expect `Hint 1/2`.
   5. Send `cat /root/READ-ME-FIRST.txt\r` → expect the literal note marker line `READ-ME-FIRST` content (regex `sealing stone|LOCKED`).
   6. Send `escape\r` → expect one CLOSED lock message (nothing fixed yet).
   7. Send the two exact fix commands from issue 30: `rm /var/dungeon/LOCKED\r` then
      `echo lit > /var/dungeon/torch\r`; send `escape\r` → expect `ESCAPED` victory banner.
   8. Process exits 0. Then run `debugdungeon list` (no pty needed) → stdout shows `welcome-cell` cleared glyph; `debugdungeon status` exits 5.
   9. Capture the run's container ID during the test (label `com.debugdungeon.run` filter) and
      assert THAT container is gone after victory — never assert on or clean the daemon's global
      managed set (a developer machine may have unrelated debugdungeon containers).
2. Robustness: every expect step logs the full transcript on failure; ANSI stripped before regex
   matching (helper reusing textsafe in test).
3. CI: job `e2e` added to `.github/workflows/scenarios.yml` (same triggers/hardening as issue 29:
   PRs touching `scenarios/**` or engine paths, weekly cron, manual dispatch), `ubuntu-latest`
   (Docker preinstalled); SHA-pinned actions; `permissions: contents: read`.
4. Local: `make e2e` documented in CONTRIBUTING (append section — small doc edit allowed here).

## Acceptance Criteria

- [ ] Test passes locally (documented run) and in CI on ubuntu-latest.
- [ ] Failure mode proven: sabotage expectation (temporarily) → transcript dump appears in test output (include example in PR description, then revert).
- [ ] Leftover hygiene: after a pass, the captured run container is gone; after an injected
  mid-test abort (subtest that kills the play process after the banner, gated by env
  `DD_E2E_INJECT_ABORT=1`), `t.Cleanup` removes ONLY the captured container ID via the Docker API
  — the suite passes and unrelated managed containers are untouched.
- [ ] Wall time < 8 min on CI.

## Validation

CI run link on the PR; local transcript attached.

## Dependencies

02, 21, 22, 23 (hint), 27 (list/status), 30.

## Non-goals

Multi-scenario E2E sweeps (harness 29 owns content coverage), macOS CI E2E (Docker unavailable on GH macOS runners — documented).

## Design References

DESIGN §12 (E2E row), §3.2 transcript; ADR-001 (test-only dep budget note), ADR-003.
