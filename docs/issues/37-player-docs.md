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
- Not: authoring docs (58), website, changes to design docs

## Detailed Requirements

1. `README.md` structure:
   - Hero: one-paragraph pitch + the demo GIF.
   - Quickstart: brew install → `debugdungeon doctor` → `play welcome-cell` (3 commands, copy-paste).
   - "What it is": floors table (from DESIGN §3.5, summarized), 60-second gameplay explanation.
   - **Security & privacy box** (verbatim commitments, each linking its enforcing artifact):
     runs rooms as hardened local containers (link DESIGN §7.4 + the profile golden test from 35);
     **no implicit network I/O by DebugDungeon** — Docker pulls pinned base images on first build,
     and that is the only network activity in the MVP (ADR-007 / DESIGN §10.8 wording, NOT "zero
     network calls"); uninstall leaves nothing behind except what `clean --all` + state-dir removal covers.
   - Requirements: Docker Desktop/OrbStack/colima (macOS), docker-ce (Linux), WSL2 note (KU-3).
   - Install alternatives: tarball + checksum verify, `gh attestation verify` one-liner, `go install`.
   - Comparison sentence(s) limited to the three named adjacent products in
     docs/research/comparative-landscape.md (SadServers, OverTheWire, iximiuz Labs) and ONLY after
     re-verifying each claim against the product's current public site — per-claim verification
     (URL + date + what was checked) recorded in the PR description.
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

- [ ] Fresh-eyes test: following ONLY the README in a clean environment (fresh macOS VM, pristine
  macOS user account, or clean Linux container/VM with Docker — any one; environment named in the
  evidence) from install through welcome-cell victory; transcript or recording attached.
- [ ] GIF renders on GitHub, ≤ 3 MB; tape file committed and re-runnable.
- [ ] Security box claims each link to their enforcing artifact (profile golden test, ADR-007, attestation docs).
- [ ] PLAYING.md covers every §5.1 MVP command; uninstall section matches §8.1 paths exactly.
- [ ] Any comparative claim carries a verification note in the PR.

## Validation

Markdown link check (lychee or manual); fresh-environment quickstart evidence; `vhs tools/demo.tape` reproduces the GIF.

## Dependencies

21–28, 30, 35 (security-gate artifacts the README links), 36.

## Non-goals

Japanese translation (v2 nice-to-have), docs website, marketing site.

## Design References

DESIGN §1, §3, §5.1, §8.1, §13; ADR-002, ADR-007; research doc (claims caveat).
