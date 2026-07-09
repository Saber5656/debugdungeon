# Title

Pre-release security audit execution (v1.0.0 gate)

## Summary

Execute a written audit of the MVP against the threat model before tagging v1.0.0, producing
`docs/security/audit-v1.0.md` with evidence for every claim. This is a blocking release gate.

## Context

DESIGN §10 makes commitments (fixed profile, sanitization, zero network, supply chain). This issue
verifies them end-to-end as a checklist with evidence, catching drift between design and code.

## Scope

- `docs/security/audit-v1.0.md` (the executed audit)
- Small fixes may be split into follow-up issues; only trivial doc fixes may ride along
- Not: new security features

## Detailed Requirements

Execute and document (each item: check → method → evidence link/output → verdict):

1. **TB2 profile**: security-regression suite (35) green on latest main; paste run link; manually
   `docker inspect` one live room on macOS+OrbStack or Docker Desktop AND Linux docker-ce — outputs archived.
2. **TB3 inputs**: validator corpus green; manual attempt: craft a hostile scenario (ANSI lore,
   symlink, `..` hint path, network:host) → `scenario` load rejects each (transcript).
3. **ANSI/terminal**: play a room with hostile `MSG:` and hint content (temporary local fixture) —
   captured terminal shows sanitized output (asciinema/script evidence).
4. **Zero network (ADR-007)**: run `list/status/map/hint/solution/version/doctor` (docker stopped
   where possible) under `strace -f -e trace=network` (Linux) — no non-docker-socket connects;
   document macOS spot-check via Little Snitch or `nettop` best-effort.
5. **State files**: perms audit (`find state root -type f -not -perm 0600`, dirs 0700); corrupt-file
   quarantine behaves per §8.4 (manual test).
6. **Supply chain**: `govulncheck` clean (or triaged); `go mod graph | wc -l` reviewed, direct deps
   match ADR-001 budget; base image digest current (< 90 days) or rotation chore filed; actions
   SHA-pin audit (`grep -R "uses:" .github/workflows` — no floating tags); release rc artifacts:
   checksum re-verified + `gh attestation verify` passes (from 36 evidence or fresh).
7. **Repo settings**: re-run every verify command from `docs/security/repo-settings.md` (38).
8. **Docs honesty**: README security box claims each verified above; SECURITY.md reporting channel
   tested (send + close a dummy private report).
9. **Residual risks** section: restate accepted risks (kernel isolation limits, PTY exemption,
   exec-timeout orphans KU/19, Podman untested KU-2, WSL2 status KU-3) — no silent acceptance.
10. Verdict block: PASS/FAIL per section + overall; FAIL items become blocking issues; the v1.0.0
    tag may only be cut when overall PASS is recorded.

## Acceptance Criteria

- [ ] `docs/security/audit-v1.0.md` complete: 10 sections, each with dated evidence.
- [ ] All verdicts PASS, or failing items linked to filed blocking issues and re-audited after fix.
- [ ] Audit performed on the release-candidate commit (hash recorded), not an older tree.
- [ ] Maintainer sign-off line (name + date) present.

## Validation

The document itself is the artifact; PR review confirms every evidence link resolves. Tag v1.0.0
references the audit commit in the release notes.

## Dependencies

35, 36, 37, 38.

## Non-goals

External penetration test (v2 consideration), formal threat-modeling workshops, pack-install audit (Wave 8 repeats this exercise as part of 61).

## Design References

DESIGN §10 (all), §12, §16; ADR-002, ADR-007; ISSUE_PLAN §6.4.
