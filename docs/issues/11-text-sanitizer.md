# Title

Terminal sanitizer for untrusted scenario/container text

## Summary

Implement `internal/textsafe`: the single sanitization chokepoint through which every
scenario-sourced or container-emitted string must pass before reaching the user's terminal
(outside the raw shell session).

## Context

DESIGN §10.5: hostile content can attack the player's terminal with escape sequences (title/
clipboard OSC, spoofing, bidi tricks). This is a hard security requirement at TB3 and a
prerequisite for helper injection (18), lock messages (19), hints (23), and solutions (25).

## Scope

- `internal/textsafe/sanitize.go` + adversarial corpus tests
- Not: markdown styling (consumer issues), the raw PTY session (exempt by design)

## Detailed Requirements

1. `Sanitize(s string, maxRunes int) string` — pipeline, in this exact order (ESC parsing FIRST,
   so sequence payloads are consumed as sequences rather than surviving generic control stripping):
   1. Enforce valid UTF-8: invalid bytes dropped (decode loop; do not insert U+FFFD).
   2. Parse and remove ESC-introduced sequences entirely: CSI (`ESC [` … final byte 0x40–0x7E),
      OSC (`ESC ]` … terminated by BEL or ST, including unterminated → strip to end), DCS/SOS/PM/APC
      (`ESC P/X/^/_` … ST), two-char sequences (`ESC` + single byte). A bare trailing ESC is
      dropped. Also parse C1-form CSI/OSC (single bytes U+009B / U+009D) as sequence introducers.
   3. Remove remaining control runes: C0 except `\n`/`\t`, DEL (U+007F), and C1 (U+0080–U+009F).
   4. Remove Unicode bidi/format controls: U+200E, U+200F, U+202A–U+202E, U+2066–U+2069, U+2028, U+2029, U+00AD.
   5. Truncate to `maxRunes` runes; if truncated, the output's final rune is `…` and total length
      is ≤ `maxRunes` runes (the ellipsis counts). `maxRunes <= 0` → returns `""`.
2. `SanitizeInline(s string, maxRunes int) string` — same pipeline with an added step between 4
   and 5: every run of `\n`/`\t` (and resulting multi-spaces) collapses to a single space
   (for table cells / titles).
3. Zero allocations is not required; correctness and auditability are. No regex for ESC parsing —
   explicit state machine (regex misses unterminated-OSC edge cases).
4. Document the exemption in `internal/textsafe/doc.go` (package comment): bytes flowing inside
   the interactive PTY session (17) are NOT sanitized (the player asked for a real terminal into
   the sandbox — DESIGN §10.2 TB2 residual risk); all other sinks MUST use this package.
5. Adversarial corpus: expressed as Go table-test cases (input as Go string literals with escape
   sequences — no binary fixture files; a small `testdata/seed/` dir holds fuzz seeds only) and
   must include:
   color CSI, cursor-move CSI, `OSC 0` title set, `OSC 8` hyperlink, `OSC 52` clipboard write,
   unterminated OSC, DCS passthrough, `\r` overwrite trick, C1 CSI (0x9B), RTL override sandwich,
   zero-width joiner flood (length cap), invalid UTF-8 overlong sequence, 1 MiB input (cap performance).

## Acceptance Criteria

- [ ] Every corpus case yields output whose **decoded runes** contain no: runes < U+0020 (except `\n`,`\t`), U+007F–U+009F, ESC, or the listed bidi/format runes. (Rune-level assertion — raw byte scans would false-positive on UTF-8 continuation bytes.)
- [ ] Idempotence test: `Sanitize(Sanitize(x)) == Sanitize(x)` over the corpus.
- [ ] Truncation: output ≤ maxRunes runes, ends with `…` when truncated, never splits a rune; `maxRunes<=0` → "".
- [ ] Fuzz smoke in normal CI (`-fuzz=FuzzSanitize -fuzztime=5s`): output always passes the rune-class assertions; seed corpus committed. (The 60s weekly fuzz job is OWNED BY issue 35 — not this issue.)

## Validation

`go test ./internal/textsafe/...` green including the fuzz seed run (`-run FuzzSanitize -fuzztime=5s` smoke in normal CI).

## Dependencies

01.

## Non-goals

HTML/markdown rendering, width-aware truncation (東アジア幅), locale handling.

## Design References

DESIGN §10.5, §10.4; threat A1 (DESIGN §10.1).
