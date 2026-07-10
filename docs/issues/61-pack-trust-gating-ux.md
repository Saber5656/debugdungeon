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
   - **Capability delta vs defaults** (the heart): per scenario, only NON-default asks, rendered
     against the explicit defaults table (from DESIGN §6.2: memory 512, cpus 1.0, pids 256, no
     tmpfs, timeout 10, network none — table restated in the disclosure code as constants), sorted
     by scenario id then field name. Base-image lines: parse each Dockerfile textually — every
     `FROM` line after ARG expansion of the leading `ARG BASE=` if present; multi-stage → list ALL
     FROM lines; unparseable/missing Dockerfile is impossible post-validation (SV010) but render
     `(unreadable)` defensively; per-line `digest-pinned: yes/no` (`@sha256:` present).
   - Warning block (fixed text, DESIGN §10.6/§10.2 TB2): what installing means —
     (a) first play runs the pack's **Dockerfile in the Docker daemon's build environment, which
     has NETWORK ACCESS** (base pulls, package installs) unlike the runtime profile;
     (b) rooms then run inside the hardened runtime profile (link);
     (c) while you are inside a room, its programs write **directly to your terminal**
     (raw PTY — control sequences included). What it does NOT mean: no review by the project.
   - Typed-name confirm (kept from 59).
2. **First-play gate**: `play` of a pack scenario with `provenance.accepted_at == null` → show a
   3-line origin reminder (name, source, installed_at) + y/N; on yes, stamp `accepted_at`
   (RFC3339 UTC) — write = read provenance.json, set field, atomic temp+rename (C9), 0600;
   last-write-wins is acceptable for this single monotonic field (documented). Declining → play
   aborts exit 1, nothing created. Non-TTY without the bypass → abort exit 1.
   `--yes-i-trust-this-pack` bypasses for automation.
   **Ordering guarantee (DESIGN §10.6):** the gate fires BEFORE `EnsureImage` — no docker build of
   pack content ever precedes both install consent and this acceptance.
3. **Origin visibility**: `list`/`map`/`status` render pack scenarios with a dim `[pack:<name>]`
   suffix (ui helper; sanitized name already regex-bound); bundled rooms get no mark.
4. **Trust lifecycle** in `pack list` (60's column): `installed` → `accepted (first played
   <YYYY-MM-DD>)` (date rendered from the RFC3339 UTC stamp, date part only).
5. Sanitization (enumerated — every displayed pack-sourced string): pack name/version/description,
   each author, source path/URL, FROM lines, scenario titles/ids in the table → each through
   `textsafe.SanitizeInline` with caps (name 30, description 200, authors 60 each, URL/path 120,
   FROM lines 120, titles 60).
6. PLAYING.md section: how to read the disclosure, advice (prefer digest-pinned FROM, read
   Dockerfiles of small packs, `clean --all` after uninstalling).
7. AUTHORING.md: replace 58's pack-section STUB with the full section (pack layout, `pack.yaml`
   fields, distribution etiquette, what installers will see — per 58's requirement 4 handoff).

## Acceptance Criteria

- [ ] Disclosure golden tests: default-only pack (short form: "no capability requests beyond defaults") vs greedy pack (every delta rendered); hostile author string sanitized.
- [ ] First-play gate: prompts once, stamps accepted_at, never re-prompts; bypass flag works; declining aborts play cleanly (no run created, and — asserted — no image build was attempted before the gate).
- [ ] Origin marks present in list/map/status goldens; bundled rows unaffected.
- [ ] itest: full journey — install (rich disclosure) → list (installed) → first play (gate) → list (accepted).
- [ ] PLAYING.md section merged and linked from the disclosure footer text.

## Validation

`make test` + itest journey transcript in PR.

## Dependencies

21, 27, 28 (map origin marks), 58 (stub being replaced), 59, 60.

## Non-goals

Pack signing/verification (v2), reputation systems, auto-revocation.

## Design References

DESIGN §6.2 (defaults), §7.4 (fixed profile), §10.1 (A1), §10.2 (TB2 raw-PTY), §10.5, §10.6;
ADR-002 (residual-risk honesty).
