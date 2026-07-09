# Title

Scenario authoring cookbook and scenario template

## Summary

Write `docs/COOKBOOK.md` — the normative how-to for building scenarios that pass validation, the
security profile, and the solvability gate — plus a copyable `scenarios/_template/` directory.

## Context

16 of the planned issues (30–33, 40–55) are content built by (possibly weak) agents; the cookbook
is what makes them mechanical. It also encodes hard-won constraints (what breakage is impossible
under the fixed container profile) so content authors don't design infeasible rooms.

## Scope

- `docs/COOKBOOK.md`
- `scenarios/_template/` (ignored by registry per issue 10)
- Not: public community guide (58 packages this for outsiders), validator code (08)

## Detailed Requirements

`docs/COOKBOOK.md` must contain these sections with this exact substance:

1. **Anatomy**: the §6.1 directory layout, with a walkthrough of every file's purpose.
2. **Base image policy**: `debian:bookworm-slim` pinned by digest; ONE shared digest constant
   documented here and used by every bundled Dockerfile via
   `ARG BASE=debian:bookworm-slim@sha256:<digest>` + `FROM ${BASE}`; the digest value is chosen at
   issue 30 implementation time (multi-arch manifest digest — verify it resolves on amd64 AND
   arm64) and recorded here; rotation procedure (quarterly chore: bump digest, run full solvability
   matrix, ship). Alpine requires justification in the scenario issue.
3. **Boot pattern**: `ENTRYPOINT ["/dungeon-boot.sh"]`; the script starts scenario processes
   (background loops with `while true; do …; sleep …; done` supervisors where crash-respawn is
   wanted), writes `/var/dungeon/boot-ok` last, ends with `exec sleep infinity`. No systemd.
   Engine sets Docker `Init: true`. Template ships a commented reference `dungeon-boot.sh`.
4. **Feasible breakage classes** (whitelist with examples): file/dir perms & ownership; config
   file corruption; PATH/env/profile sabotage; cron (must run `cron` daemon in boot script);
   plain-process services & respawn loops; disk-full and inode exhaustion **only inside spec tmpfs
   mounts**; log noise; `/etc/hosts` + `nsswitch.conf`; localhost multi-process topologies
   (nginx→backend, app→postgres/redis on 127.0.0.1); TLS cert files; user/group membership.
5. **Infeasible under the fixed profile** (hard NO list, with the failing capability):
   iptables/nftables (NET_ADMIN), mount/remount/swap (SYS_ADMIN), `chattr +i` (LINUX_IMMUTABLE),
   kernel modules & most sysctls, time change (SYS_TIME), device nodes (MKNOD), raw sockets/ping
   puzzles (NET_RAW dropped), outbound network at runtime (network=none), systemd, sudo-based
   puzzles for non-root entry users (no-new-privileges blocks setuid) — for those, make `entry.user` root.
6. **Lock authoring rules**: read-only (must not mutate game state); fast (< timeout, default 10s);
   deterministic; end with a clear `MSG: <player-safe failure hint>` on the last stdout line when
   failing; retry-with-deadline pattern snippet for eventually-true conditions (cron scenarios);
   never reference `/dungeon` paths; assume root execution via stdin-piped `/bin/sh` (POSIX, no bashisms).
7. **solution.sh rules**: idempotent-ish, non-interactive, POSIX sh, < 300s, must leave ALL locks
   open; must not use the in-room helpers.
8. **Hints style**: 2–4 hints, escalating (concept → location → near-answer); ≤ 4 KiB each;
   markdown-lite (bold/code only); never contain literal full solution commands in hint 1–2.
9. **Lore & text**: ≤ 1500 chars lore; plain ASCII preferred; remember all text is sanitized (11).
10. **Arch neutrality**: no downloaded binaries, no arch-conditional logic; apt packages only
    (`--no-install-recommends`, `rm -rf /var/lib/apt/lists/*`); everything must pass the
    solvability matrix on amd64 + arm64 (ADR-006).
11. **Size budget**: uncompressed ≤ 500 MB (Floor 5 pre-approved to 700 MB); measure with
    `docker image inspect --format '{{.Size}}'`.
12. **Spoiler hygiene**: image must not contain hints/solutions/checks; consolidate breakage into
    one `RUN` layer; never `COPY` the scenario root.
13. **Do-not-break list**: `/dungeon` prefix reserved (engine helpers); do not remove `sh`, `cat`,
    `ls`, or the entry shell; do not fill `/` (only tmpfs mounts may be filled).
14. **Submission checklist** (copy into content PRs): validator green, `scenario test` green both
    arches, size within budget, hints escalate, solution.md has Lesson Learned, lore sanitized-safe.

`scenarios/_template/`: complete runnable example ("hello-room": one trivial lock, one hint) with
commented `scenario.yaml`, `image/Dockerfile` (ARG BASE pattern), `image/dungeon-boot.sh`,
`checks/example.sh` (MSG pattern + retry snippet), `hints/01.md`, `solution.md`, `solution.sh`.

## Acceptance Criteria

- [ ] COOKBOOK.md contains all 14 sections above with concrete commands/snippets (not prose-only).
- [ ] `_template` passes `ValidateDir` (08) with zero violations (test added in this issue).
- [ ] Template Dockerfile uses the `ARG BASE` digest pattern and builds FROM bookworm-slim (digest placeholder documented as "set in issue 30").
- [ ] Infeasible list names the exact missing capability for each item (matches DESIGN §7.4 CapAdd set).

## Validation

`go test ./internal/scenario/... -run Template` (new test validating `_template`); markdown lint
(mdl or manual) — no broken intra-repo links.

## Dependencies

07, 08.

## Non-goals

Public-facing tutorial polish (58), scaffolder automation (56), choosing the actual digest (30).

## Design References

DESIGN §6.1–6.6, §7.4, §10.3; ADR-002, ADR-005, ADR-006.
