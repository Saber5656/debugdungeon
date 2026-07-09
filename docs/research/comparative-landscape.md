# Research: Comparative Landscape

Status: knowledge-based snapshot written 2026-07-10. Product/feature claims below reflect the
author-model's knowledge (cutoff early 2026) plus a light web spot-check of ecosystem facts
(Go release line). **Re-verify any claim before using it in marketing or README comparisons**
(tracked as a task inside issue 37).

## Why this matters to the design

The concept sits between "ops troubleshooting labs" and "terminal games". The design choices it
motivated: local-first + no account (vs hosted labs), real containers (vs simulated shells),
game framing with floors/locks/lore (vs bare challenge lists), and a provable-solvability content
pipeline (a weakness observed across community challenge repos).

## Adjacent products

| Product | Model | What we learn / how we differ |
|---|---|---|
| SadServers | Hosted "fix the broken server" VMs, browser terminal, per-scenario checks | Closest concept validation. Differences: we're local/offline, OSS, no accounts, game progression. Their per-scenario "check my solution" button ≈ our locks |
| OverTheWire (Bandit…) | SSH wargames, exploitation/CTF flavored | Progression-by-level works; but focus is offense/CTF, ours is operational debugging |
| KodeKloud Engineer / labs | Hosted DevOps task labs, guided | Curriculum breadth; heavier platform, subscription, not a game |
| iximiuz Labs | Hosted container/K8s playgrounds & challenges | Strong container focus, hosted; validates appetite for infra debugging practice |
| KillerCoda / Katacoda-style | Hosted interactive scenarios in browser | Authoring-format idea (scenario-as-directory) validated; hosted-only |
| `wargames`/CTFd self-host | Self-hosted challenge platforms | Self-hosting exists but web-first and competition-oriented |
| Exercism / code katas | Fix/write code against tests | The "artifact = environment, not code" gap we fill |
| MIT "Missing Semester" etc. | Courseware | Audience overlap (learners), no interactive broken environments |

## Gap statement

As of this snapshot, no notable OSS tool offers: `brew install` → real broken Linux environments
in local containers → single-terminal game loop with locks/hints/progression → community scenario
format with an enforced solvability gate. That combination is the v1 bet.

## Risks flagged by the landscape

1. Hosted competitors iterate content faster (no local build wait) → our first-build UX and image
   size budgets matter (DESIGN §6.6, §11 F3).
2. Content quality is the moat and the treadmill → authoring kit + solvability gate are v1-full
   scope, not afterthoughts (ADR-006).
3. Docker-as-prerequisite filters out some beginners → `doctor` + install docs quality is a
   product feature, not a chore (issues 13, 37).
