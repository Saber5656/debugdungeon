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

1. **TB2 profile**: security-regression suite (35) green **on the release-candidate commit** (the
   single audited tree — every evidence item in this audit records that commit hash); paste run
   link; manually `docker inspect` one live room on macOS+OrbStack or Docker Desktop AND Linux
   docker-ce — outputs archived.
2. **TB3 inputs**: validator corpus green; manual attempt: craft a hostile scenario (ANSI lore,
   symlink, `..` hint path, network:host) → `scenario` load rejects each (transcript).
3. **ANSI/terminal**: play a room with hostile `MSG:` and hint content (temporary local fixture) —
   captured terminal shows sanitized output for every NON-raw-PTY sink (banner, hint, lock
   messages, solution, list/status/map). The raw shell itself is exempt by design (DESIGN §10.2
   TB2 residual risk) — the audit records that exemption, it does not test the impossible.
4. **No implicit network (ADR-007)**: setup = temp `DEBUGDUNGEON_HOME` with a cleared+given-up
   progress fixture (so `hint`/`solution` have state) and Docker STOPPED; run
   `strace -f -e trace=network` (Linux) over `list`, `status` (exit 5 ok), `map`, `solution
   welcome-cell`, `version`, `doctor` (exit 3 ok) — PASS = no `connect` to AF_INET/AF_INET6
   sockets; AF_UNIX connects to the Docker socket are expected/allowed for docker-touching
   commands. macOS spot-check via `nettop`/Little Snitch best-effort, documented.
5. **State files**: on BOTH a default state root and an isolated `DEBUGDUNGEON_HOME` (after one
   full play): `find <root> -type f ! -perm 600` and `find <root> -type d ! -perm 700` both empty;
   quarantine test: truncate `progress.json` mid-JSON → next `list` warns once, creates fresh,
   leaves `progress.json.corrupt-<ts>` (transcript archived).
6. **Supply chain**: `govulncheck` clean (or triaged); `go mod graph | wc -l` reviewed, direct deps
   match ADR-001 budget; base image digest equals COOKBOOK §2's documented digest AND its recorded
   selection date (kept in COOKBOOK §2) is < 90 days old, else rotation chore filed; actions
   SHA-pin audit: every non-local `uses:` in `.github/workflows/` matches `@[0-9a-f]{40}`
   (grep -E; any `@v\d`/`@main` = FAIL); release rc artifacts: checksum re-verified +
   `gh attestation verify` passes (from 36 evidence or fresh).
7. **Repo settings**: re-run every verify command from `docs/security/repo-settings.md` (38).
8. **Docs honesty**: README security box claims each verified above; SECURITY.md reporting
   channel: verify enabled via 38's API check + document the maintainer-only test procedure
   (GitHub PVR does not support disposable test reports cleanly — a live dummy report is NOT required).
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
