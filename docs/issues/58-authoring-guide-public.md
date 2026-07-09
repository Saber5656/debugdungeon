# Title

Public authoring guide (docs/AUTHORING.md)

## Summary

Write the public-facing scenario authoring guide: a narrative tutorial from `scenario init` to a
submitted room, packaging the cookbook's rules for outside contributors.

## Context

COOKBOOK (12) is the normative reference written for in-repo content issues; AUTHORING.md is the
welcoming on-ramp (DESIGN §1.3 P3 persona). Wave 8's pack path and the in-repo contribution path
both start here.

## Scope

- `docs/AUTHORING.md`; link from README + CONTRIBUTING
- Not: changing cookbook rules (any conflict = bug, cookbook wins), pack install mechanics (59 docs)

## Detailed Requirements

1. Structure:
   1. *Your first room in 20 minutes*: `scenario init` → edit lore/breakage → `validate` → `test` →
      `play` it locally (`play` on external dirs? NOT supported in v1 — playtesting happens via
      `scenario test --keep` + manual exec; state this honestly with the exec one-liner).
   2. *Anatomy deep-dive*: link-and-summarize COOKBOOK sections (no duplication of normative
      tables — link, don't fork; short summaries only).
   3. *Design a good room*: the craft chapter — one skill per room; symptom → trail → cause
      structure; hint escalation; trap-with-teaching-MSG pattern (examples: 32's 777-trap,
      41's respawn trap); lock MSG voice guide; lore tone (2–4 sentences, playful-grim, no walls of text).
   4. *The two submission paths*: (a) PR into `scenarios/` (bundled; full review + CI gates;
      checklist); (b) community pack (59–61): pack layout, `pack.yaml`, distribution etiquette,
      the trust gate users will see (set expectations: your pack is untrusted by default).
   5. *Reference card*: one-page table — commands, budgets, feasible/infeasible classes (from
      COOKBOOK §4/§5), rule-code quick list (SV001–SV025 one-liners).
2. Every command transcript must be real (copy-pasted from actual runs at writing time).
3. Tone: second-person, concise; English; assumes reader knows Linux basics but not this repo.

## Acceptance Criteria

- [ ] A fresh agent following ONLY AUTHORING.md produces a novel passing room (validate+test green) without reading COOKBOOK — evidence transcript attached.
- [ ] Zero normative duplication drift: rules appear as links/summaries with pointers, not re-stated tables (spot-check).
- [ ] README + CONTRIBUTING link to it; `scenario init` epilogue (56) points at it (already specified there — verify).
- [ ] Reference card fits one screen (~60 lines).

## Validation

Fresh-eyes authoring run (as above); markdown link check.

## Dependencies

12, 56, 57.

## Non-goals

Video tutorials, JP translation (v2), gallery/showcase page.

## Design References

DESIGN §1.3 (P3), §2.2; COOKBOOK (12); ISSUE_PLAN wave 7–8.
