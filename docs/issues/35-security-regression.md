# Title

Security regression test suite

## Summary

Turn the security model's key invariants into permanent CI gates: live container-profile
assertions, validator hostile-input corpus, sanitizer adversarial corpus + fuzz smoke, and
dependency vulnerability scanning.

## Context

DESIGN §10.3 promises the profile is enforced by "code + regression tests". This issue is the
regression half; issue 39 (audit) consumes its output as evidence.

## Scope

- `test/security/` test files (tags `itest`) + additions to existing package tests
- `.github/workflows/security.yml`
- Not: fixing findings (separate issues if any), pack-extraction corpus (59 adds its own)

## Detailed Requirements

1. **Live profile golden (TB2)** — extends 15's itest into a standalone named gate. Security
   fixture: `test/security/testdata/profile-room/` — a minimal scenario dir (C3 layout) whose
   Dockerfile installs, in THIS image only, the probe tooling: `bash`, `iproute2`, `e2fsprogs`
   (chattr), `coreutils` (nice); maximal-resources manifest incl. one tmpfs mount.
   Create the room, `ContainerInspect`, assert exactly:
   `NetworkMode=="none"`, `CapDrop==["ALL"]`, CapAdd set-equal to DESIGN §7.4 list, SecurityOpt
   contains `no-new-privileges:true`, `Privileged==false`, `Binds/Devices/DeviceRequests` empty,
   `Mounts/Tmpfs` == declared tmpfs only, `PidMode/IpcMode/UTSMode` private (empty),
   `VolumesFrom` empty, `Config.Volumes` empty, `PublishAllPorts==false`, `PortBindings` empty,
   `PidsLimit/Memory/NanoCPUs` match clamped spec, `Init==true`, `RestartPolicy=="no"`,
   `NetworkSettings.Networks` contains only the `none` network.
   Network-egress probe: `bash -c 'echo x > /dev/tcp/1.1.1.1/80'` (bash builtin — fixture installs
   bash) must exit non-zero.
2. **In-container escalation probes** (all must exit non-zero, asserting the profile bites; each
   probe names its blocked capability in the test name): `mount -o remount,rw /` (SYS_ADMIN),
   `iptables -L` → not installed; use `ip link set lo down` (NET_ADMIN), `chattr +i /etc/hostname`
   (LINUX_IMMUTABLE), `nice -n -10 true` (SYS_NICE). Non-zero exit suffices; stderr logged.
3. **Validator hostile corpus (TB3)**: ensure 08's corpus includes and CI-runs: symlink escape,
   `..` traversal in every path field, oversize file/tree, >400 files, network `host|bridge|internal`,
   privileged-looking unknown fields (`privileged: true`, `cap_add: [...]`, `binds: [...]`) — each
   rejected (unknown-field strictness proves capability creep is structurally impossible).
4. **Sanitizer gate (TB3)**: run 11's corpus + `-fuzz=FuzzSanitize -fuzztime=60s` in the weekly job
   (5s smoke on PR).
5. **Dependency scanning**: `govulncheck ./...` job — tool version pinned per CONVENTIONS C6
   (`go run golang.org/x/vuln/cmd/govulncheck@<pinned>`, resolved at implementation, recorded with
   a dated comment in the workflow); failure policy: fails the job; a documented ignore requires a
   linked issue (policy text in workflow comment).
6. `security.yml`: PR trigger on engine paths + weekly cron; same hardening rules as 02;
   `permissions: contents: read`.
7. A `docs/security/README.md` stub mapping each gate to DESIGN §10 rows (audit 39 fills results).

## Acceptance Criteria

- [ ] All gates green on current code; sabotage matrix executed and evidenced (one-line temporary
      patches, evidence in PR, reverted) — each row names the expected RED gate:
      add `NET_ADMIN` to CapAdd → profile golden + `ip link` probe; set `NetworkMode=bridge` →
      profile golden + egress probe; add a Bind mount → profile golden; disable a sanitizer step →
      textsafe corpus; allow unknown manifest fields → validator corpus case.
- [ ] govulncheck job present, pinned, passing.
- [ ] Weekly cron includes fuzz 60s + full corpus.
- [ ] `docs/security/README.md` maps gates → DESIGN §10.1–10.5 items with file/test names.

## Validation

CI links (PR run + a manual cron dispatch); sabotage evidence screenshots/logs in PR.

## Dependencies

02, 08, 11, 15, 19.

## Non-goals

Pack/archive attack corpus (59), secrets scanning config (38), penetration testing (39 checklist references external effort as future).

## Design References

DESIGN §7.4, §10.1–10.5, §12; ADR-002.
