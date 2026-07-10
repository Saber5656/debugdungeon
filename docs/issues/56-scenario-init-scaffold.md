# Title

`scenario init` scaffolder

## Summary

Implement `debugdungeon scenario init <dir>`: generate a new, immediately-valid scenario skeleton
from the `_template`, parameterized by flags, with next-step guidance.

## Context

Wave 7 opens authoring to third parties (DESIGN §2.2). The scaffold must produce something that
passes `scenario validate` AND `scenario test` out of the box, so authors start from green.

## Scope

- `internal/cli/scenario_init.go`, template rendering helper + tests
- Not: validate/test commands (57), public guide (58)

## Detailed Requirements

1. Command: `debugdungeon scenario init <dir>` with flags:
   `--id` (default: `path.Base(dir)`; validated against SV002 regex pre-flight),
   `--title` (default `The Nameless Room`), `--floor` (default 1), `--difficulty` (default 1),
   `--topics` (csv, default `basics`), `--time` (default 15).
2. Source of truth: the embedded `scenarios/_template` (10's embed) — recursive copy (template
   contains only regular files by construction; every file is text and gets substitution), dirs
   0755 / files 0644, deterministic walk order. Placeholder substitution (`{{ID}}`, `{{TITLE}}`,
   `{{FLOOR}}`, `{{DIFFICULTY}}`, `{{TOPICS_YAML}}`, `{{TIME}}`) via plain string replacement
   (no template engine — predictable for authors).
3. Flag validation BEFORE any write, mirroring validator ranges: id per SV002 regex; floor 1–6;
   difficulty 1–5; time 5–120; topics each per SV006 regex. `--title` is YAML-escaped by
   rendering as a double-quoted YAML scalar (quotes/backslashes escaped) — flag input can never
   produce invalid YAML or smuggle extra keys.
4. Destination semantics: parent dirs created (`MkdirAll`); refuse an existing non-empty dir or an
   existing non-directory (exit 1, message); empty existing dir is fine. On any error mid-copy,
   remove everything this run created (track created paths; never remove a pre-existing dir itself).
5. Output files: full DESIGN §6.1 layout, plus `NOTES.md` — explicitly ALLOWED at scenario root
   (extra files pass SV022/SV024; name is compliant) — containing the author checklist (from
   COOKBOOK §14) and the immediate next commands.
6. Epilogue print:
   ```
   Room scaffolded at <dir>.
   Next: 1) edit scenario.yaml + image/   2) debugdungeon scenario validate <dir>
         3) debugdungeon scenario test <dir>   (requires Docker)
   Guide: https://github.com/Saber5656/debugdungeon/blob/main/docs/AUTHORING.md
   ```
   (URL, not a relative path — the scaffold runs from an installed binary with no repo checkout;
   the page exists once 58 lands, acceptable forward reference within the same wave.)
7. Generated scenario must pass `ValidateDir` and the solvability harness (29) unmodified —
   asserted on the scaffolded COPY (not just the template).

## Acceptance Criteria

- [ ] `scenario init /tmp/x --id my-room` → `ValidateDir` zero violations on the result; harness passes it (itest).
- [ ] All placeholders substituted (grep test: no `{{` remains); topics csv → correct YAML list.
- [ ] Refusal paths (existing dir, bad id) covered with correct exit codes.
- [ ] Scaffold works from the installed binary without network/repo checkout (embed-sourced).

## Validation

`go test` + one itest scaffolding→harness round trip; transcript in PR.

## Dependencies

07, 08, 10 (embedded template access), 12, 29 (harness assertion).

## Non-goals

Interactive wizard, pack scaffolding (59's format docs cover it), updating existing scenarios.

## Design References

DESIGN §5.1, §6.1–6.2, §6.4, §10.4 (TB3 — flag inputs), §7.2; COOKBOOK (12); ISSUE_PLAN wave 7.
