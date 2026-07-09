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

1. `Sanitize(s string, maxRunes int) string` — pipeline, in order:
   1. Enforce valid UTF-8: invalid bytes dropped (decode loop; do not insert U+FFFD).
   2. Remove all C0 controls except `\n` and `\t`; remove DEL (0x7F); remove all C1 (0x80–0x9F).
   3. Remove ESC-introduced sequences entirely: CSI (`ESC [` … final byte 0x40–0x7E), OSC
      (`ESC ]` … terminated by BEL or ST, including unterminated → strip to end), DCS/SOS/PM/APC
      (`ESC P/X/^/_` … ST), two-char sequences (`ESC` + single byte). A bare trailing ESC is dropped.
   4. Remove Unicode bidi/format controls: U+200E, U+200F, U+202A–U+202E, U+2066–U+2069, U+2028, U+2029, U+00AD.
   5. Truncate to `maxRunes` runes; if truncated, append `…`.
2. `SanitizeInline(s string, maxRunes int) string` — same, but `\n`/`\t` become single spaces
   (for table cells / titles).
3. Zero allocations is not required; correctness and auditability are. No regex for ESC parsing —
   explicit state machine (regex misses unterminated-OSC edge cases).
4. Document the exemption: bytes flowing inside the interactive PTY session (17) are NOT sanitized
   (the player asked for a real terminal into the sandbox); all other sinks MUST use this package.
5. Adversarial corpus (committed as `testdata/hostile.txt` cases + table tests) must include:
   color CSI, cursor-move CSI, `OSC 0` title set, `OSC 8` hyperlink, `OSC 52` clipboard write,
   unterminated OSC, DCS passthrough, `\r` overwrite trick, C1 CSI (0x9B), RTL override sandwich,
   zero-width joiner flood (length cap), invalid UTF-8 overlong sequence, 1 MiB input (cap performance).

## Acceptance Criteria

- [ ] Every corpus case yields output free of: bytes < 0x20 (except `\n`,`\t`), 0x7F–0x9F, ESC, listed bidi/format runes.
- [ ] Idempotence test: `Sanitize(Sanitize(x)) == Sanitize(x)` over the corpus.
- [ ] Truncation adds `…` and never splits a rune.
- [ ] Fuzz test (`go test -fuzz=FuzzSanitize -fuzztime=30s` in CI weekly job note): output always passes the byte-class assertions; add seed corpus.

## Validation

`go test ./internal/textsafe/...` green including the fuzz seed run (`-run FuzzSanitize -fuzztime=5s` smoke in normal CI).

## Dependencies

01.

## Non-goals

HTML/markdown rendering, width-aware truncation (東アジア幅), locale handling.

## Design References

DESIGN §10.5, §10.4; threat A1 (DESIGN §10.1).
