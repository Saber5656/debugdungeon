# Title

Community pack trust gating UX

## Summary

Complete the TB4 consent layer: rich capability disclosure at install, first-play origin warning
per pack, visible origin marks across `list`/`status`/`map`, and the trust-state lifecycle in
provenance.

## Context

DESIGN §10.6 commits to explicit, informed consent before third-party code builds/runs on the
player's machine. 59/60 implemented mechanics + a minimal confirm; this issue makes the gate
genuinely informative and hard to click through blindly.

## Scope

- `internal/pack/disclosure.go`, touches to `internal/cli/{play,list,status}.go`, `internal/ui`
- A short user-doc section in PLAYING.md ("Installing community packs safely")
- Not: signature verification (v2/KU-6 adjacent), sandbox changes (profile already fixed)

## Detailed Requirements

1. **Install-time disclosure** (replaces 59's minimal block) — a boxed report:
   - Identity: name, version, authors, source (full URL/path + commit/sha256).
   - Content: scenario table (id, floor, ★, topics, time).
   - **Capability delta vs defaults** (the heart): per scenario, only NON-default asks —
     resources above defaults, tmpfs mounts (path+size), lock timeouts > 10s; plus base image
     FROM lines with digest-pinned? yes/no flag.
   - Warning block (fixed text, DESIGN §10.6/§10.2 TB2): what installing means —
     (a) first play runs the pack's **Dockerfile in the Docker daemon's build environment, which
     has NETWORK ACCESS** (base pulls, package installs) unlike the runtime profile;
     (b) rooms then run inside the hardened runtime profile (link);
     (c) while you are inside a room, its programs write **directly to your terminal**
     (raw PTY — control sequences included). What it does NOT mean: no review by the project.
   - Typed-name confirm (kept from 59).
2. **First-play gate**: `play` of a pack scenario with `provenance.accepted_at == null` → show a
   3-line origin reminder (name, source, installed_at) + y/N; on yes, stamp `accepted_at` (once
   per pack). `--yes-i-trust-this-pack` bypasses for automation.
   **Ordering guarantee (DESIGN §10.6):** the gate fires BEFORE `EnsureImage` — no docker build of
   pack content ever precedes both install consent and this acceptance.
3. **Origin visibility**: `list`/`map`/`status` render pack scenarios with a dim `[pack:<name>]`
   suffix (ui helper; sanitized name already regex-bound); bundled rooms get no mark.
4. **Trust lifecycle** in `pack list` (60's column): `installed` → `accepted (first played <date>)`.
5. All disclosure text fields flow through 11 (authors/description are hostile-capable).
6. PLAYING.md section: how to read the disclosure, advice (prefer digest-pinned FROM, read
   Dockerfiles of small packs, `clean --all` after uninstalling).

## Acceptance Criteria

- [ ] Disclosure golden tests: default-only pack (short form: "no capability requests beyond defaults") vs greedy pack (every delta rendered); hostile author string sanitized.
- [ ] First-play gate: prompts once, stamps accepted_at, never re-prompts; bypass flag works; declining aborts play cleanly (no run created, and — asserted — no image build was attempted before the gate).
- [ ] Origin marks present in list/map/status goldens; bundled rows unaffected.
- [ ] itest: full journey — install (rich disclosure) → list (installed) → first play (gate) → list (accepted).
- [ ] PLAYING.md section merged and linked from the disclosure footer text.

## Validation

`make test` + itest journey transcript in PR.

## Dependencies

21, 27, 59, 60.

## Non-goals

Pack signing/verification (v2), reputation systems, auto-revocation.

## Design References

DESIGN §10.6, §10.1 (A1), §10.5; ADR-002 (residual-risk honesty).
