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
   invoked command is `version` AND last check ≥ 24h ago. Cache file
   `<state>/update-check.json` (0600, atomic temp+rename per C9): `{schema_version:1,
   last_checked_at: RFC3339, latest_seen: string}`; unreadable/corrupt/newer-schema → treat as
   "never checked", overwrite on next run (no quarantine — it's a disposable cache). Clock going
   backwards (last_checked_at in the future) → treat as due.
2. Request: single `GET https://api.github.com/repos/Saber5656/debugdungeon/releases/latest` via a
   dedicated `http.Client{Timeout: 2s, CheckRedirect: refuse cross-host + cap 3}` with
   `Transport{DisableKeepAlives:true, DisableCompression:false}`; only header set is
   `User-Agent: debugdungeon/<version>` (Go adds Host/Accept-Encoding automatically — "no other
   headers" means we set none else). Parse only `{tag_name string, prerelease bool}`; ignore
   entries with `prerelease==true`. Any failure / invalid JSON / non-semver `tag_name` → silent
   (debug log only), cache timestamp still updated (don't retry-spam).
3. Compare semver (strip `v`; the running `version.Version` may be `dev`/dirty → never notify in
   that case). Newer stable → single stderr line after version output:
   `update available: v1.3.0 → v1.4.1 (brew upgrade debugdungeon)`. Respects no-color.
4. Hard guarantees (tested):
   - Test seam: `updatecheck.Check(ctx, deps, transport http.RoundTripper)`; production passes the
     dedicated client's transport, tests pass a fake. There is NO way for user config/env to point
     the check at a different endpoint (the URL is a compile-time constant) — so a "local httptest
     override" exists ONLY in tests via the injected transport, never in the shipped binary.
   - With default config: zero network calls process-wide for EVERY command — assert by walking
     the cobra command tree in a test with a transport that `t.Fatal`s if invoked.
   - `DEBUGDUNGEON_NO_NETWORK=1` beats config true.
   - Only `version` wires updatecheck (arch/go-list test: no other cli command file imports it).
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

ADR-007 (the carve-out definition); DESIGN §2.3, §5.1, §8.1, §8.4, §10.8; ISSUE_PLAN §7 (deferred/boundary).
