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
2. Source of truth: the embedded `scenarios/_template` (10's embed) — files copied with
   placeholder substitution (`{{ID}}`, `{{TITLE}}`, `{{FLOOR}}`, `{{DIFFICULTY}}`, `{{TOPICS_YAML}}`,
   `{{TIME}}`) via plain string replacement (no template engine — predictable for authors).
3. Refuses: existing non-empty `<dir>` (exit 1), invalid id (exit 4 with the SV002 regex shown).
4. Output files: full DESIGN §6.1 layout, plus `NOTES.md` (untracked-by-schema? NO — unknown files
   are allowed by SV-rules only if… SV022/SV024 permit extra files; keep NOTES.md name-compliant)
   containing the author checklist (from COOKBOOK §14) and the immediate next commands.
5. Epilogue print:
   ```
   Room scaffolded at <dir>.
   Next: 1) edit scenario.yaml + image/   2) debugdungeon scenario validate <dir>
         3) debugdungeon scenario test <dir>   (requires Docker)
   Guide: docs/AUTHORING.md
   ```
6. Generated scenario must pass `ValidateDir` and the solvability harness unmodified (template's
   hello-room is solvable by construction — 12's AC already guarantees; test re-asserts on the scaffolded copy).

## Acceptance Criteria

- [ ] `scenario init /tmp/x --id my-room` → `ValidateDir` zero violations on the result; harness passes it (itest).
- [ ] All placeholders substituted (grep test: no `{{` remains); topics csv → correct YAML list.
- [ ] Refusal paths (existing dir, bad id) covered with correct exit codes.
- [ ] Scaffold works from the installed binary without network/repo checkout (embed-sourced).

## Validation

`go test` + one itest scaffolding→harness round trip; transcript in PR.

## Dependencies

07, 08, 12 (10 for embedded template access).

## Non-goals

Interactive wizard, pack scaffolding (59's format docs cover it), updating existing scenarios.

## Design References

DESIGN §5.1, §6.1; COOKBOOK (12); ISSUE_PLAN wave 7.
