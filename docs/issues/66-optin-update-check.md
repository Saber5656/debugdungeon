# Title

Opt-in update check (privacy-strict)

## Summary

Implement the ONLY sanctioned network feature of v1-full: an explicitly opt-in, rate-limited,
GET-only check against GitHub Releases that prints a one-line notice when a newer version exists.

## Context

ADR-007 permits exactly this shape: default OFF, config-gated, no identifiers, result cached.
Discovery otherwise relies on brew upgrade. This issue must be paranoid-by-construction because it
pierces the zero-network posture.

## Scope

- `internal/updatecheck/updatecheck.go`, wiring into `version` command only + tests
- ADR-007 compliance additions to the security regression suite (35 extension)
- Not: auto-download, auto-update, nag prompts

## Detailed Requirements

1. Gate chain (ALL must hold, checked in order, short-circuit silent):
   `config.update_check == true` (05; default false) AND env `DEBUGDUNGEON_NO_NETWORK` unset AND
   invoked command is `version` AND last check ≥ 24h ago (cache file `<state>/update-check.json`:
   `{last_checked_at, latest_seen}`).
2. Request: single `GET https://api.github.com/repos/Saber5656/debugdungeon/releases/latest`,
   timeout 2s, no redirects beyond 3, `User-Agent: debugdungeon/<version>`, NO other headers,
   no etag persistence beyond the cache file above (keep it dumb). Any failure → silent (debug log only), cache timestamp updated (don't retry-spam).
3. Compare semver (strip `v`); newer → single stderr line after version output:
   `update available: v1.3.0 → v1.4.1 (brew upgrade debugdungeon)`. Never colors mandatory, respects no-color.
4. Hard guarantees (tested):
   - With default config: zero network calls process-wide for EVERY command (assert via injected
     RoundTripper that fails the test if used — wire http.Client injection; default client
     constructed ONLY inside updatecheck behind the gate).
   - `DEBUGDUNGEON_NO_NETWORK=1` beats config true.
   - Non-`version` commands never construct the client (arch test: only `version` imports updatecheck… enforced via depguard or a go-list test).
5. Docs: PLAYING.md FAQ entry + config.yaml comment; ADR-007 gets a one-line "implemented by
   issue 66 within the carve-out" annotation (edit allowed).
6. 35 extension: security suite gains the "default = zero egress" test (the strace audit in 39
   remains the release-time double check; if v1.0.0 already shipped, this lands in the next audit doc).

## Acceptance Criteria

- [ ] Gate-chain table tests (each gate independently blocks; all-pass path fires exactly one request via fake RoundTripper).
- [ ] 24h rate limit honored across process restarts (cache file); failure path silent + cached.
- [ ] Notice line golden (newer / equal / older / prerelease-on-latest handled: prerelease ignored).
- [ ] Zero-egress default proven by the injected-transport test across all commands (test walks cobra command tree).
- [ ] `version` output unchanged when check disabled (byte-identical golden).

## Validation

`go test ./internal/updatecheck/...` + the cross-command zero-egress test; manual: enable in
config, run version twice (one request, then cached) with a local httptest override.

## Dependencies

04, 05 (35 for the suite extension).

## Non-goals

Auto-update, checking on every command, telemetry of ANY kind, canary/notification channels.

## Design References

ADR-007 (the carve-out definition); DESIGN §2.3, §10.8; ISSUE_PLAN §7 (v2 boundary).
