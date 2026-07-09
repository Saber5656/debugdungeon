# Title

Player documentation: README overhaul, playing guide, demo

## Summary

Write the public-facing docs that sell and explain the game: README with quickstart + demo GIF,
`docs/PLAYING.md` player guide, and troubleshooting/uninstall coverage.

## Context

Docs are the top of the funnel and part of the security posture (honest about what runs where —
ADR-002/007). Research doc risk 3: Docker-prereq friction must be absorbed by documentation quality.

## Scope

- `README.md` (rewrite), `docs/PLAYING.md`, `tools/demo.tape` (vhs script) + committed GIF
- Small `docs/` index touch-ups
- Not: authoring docs (58), website

## Detailed Requirements

1. `README.md` structure:
   - Hero: one-paragraph pitch + the demo GIF.
   - Quickstart: brew install → `debugdungeon doctor` → `play welcome-cell` (3 commands, copy-paste).
   - "What it is": floors table (from DESIGN §3.5, summarized), 60-second gameplay explanation.
   - **Security & privacy box** (verbatim commitments): runs rooms as hardened local containers
     (link DESIGN §7.4); CLI makes zero network calls (ADR-007); first build pulls a pinned base
     image via Docker; uninstall leaves nothing behind except what `clean --all` + state-dir removal covers.
   - Requirements: Docker Desktop/OrbStack/colima (macOS), docker-ce (Linux), WSL2 note (KU-3).
   - Install alternatives: tarball + checksum verify, `gh attestation verify` one-liner, `go install`.
   - Comparison sentence(s) with adjacent tools ONLY after re-verifying claims (research doc
     caveat) — verification results recorded in PR.
   - Contributing/scenario-authoring pointers, license badge, CI badges.
2. `docs/PLAYING.md`:
   - Full command reference (mirrors DESIGN §5.1 table, player-voice).
   - The in-room protocol (`escape`/`hint`/`giveup`, pausing, second-terminal `check`).
   - Progression rules (§3.5), scoring fields, free-roam mode.
   - FAQ: first build slow; offline play; Ctrl-C semantics; cwd reset after hint (§3.3 quirk);
     "is this safe?" (profile summary + link); disk usage & `clean --all`; WSL2 setup steps.
   - Uninstall: brew uninstall, `clean --all`, state dir paths per OS (§8.1 table verbatim).
3. `tools/demo.tape` (vhs): scripted 30–45s: `list` → `play welcome-cell` → hint → fix → `escape`
   → victory → `map`. GIF ≤ 3 MB committed at `docs/assets/demo.gif`. vhs is a dev-only tool
   (not a project dependency); document install line in the tape header.
4. All shell snippets copy-paste-tested; README examples must not drift from actual CLI output
   (paste from real runs).

## Acceptance Criteria

- [ ] Fresh-eyes test: a person/agent following only README on a clean macOS VM reaches victory in welcome-cell (transcript or recording as evidence).
- [ ] GIF renders on GitHub, ≤ 3 MB; tape file committed and re-runnable.
- [ ] Security box claims each link to their enforcing artifact (profile golden test, ADR-007, attestation docs).
- [ ] PLAYING.md covers every §5.1 MVP command; uninstall section matches §8.1 paths exactly.
- [ ] Any comparative claim carries a verification note in the PR.

## Validation

Markdown link check (lychee or manual); fresh-environment quickstart evidence; `vhs tools/demo.tape` reproduces the GIF.

## Dependencies

21–28, 30, 36.

## Non-goals

Japanese translation (v2 nice-to-have), docs website, marketing site.

## Design References

DESIGN §1, §3, §5.1, §8.1, §13; ADR-002, ADR-007; research doc (claims caveat).
