# Title

Scenario: dns-demon (Floor 4 — name resolution sabotage)

## Summary

Floor-4 room: the courier app can't reach `vault.internal` because `/etc/hosts` is poisoned and
`/etc/nsswitch.conf`'s hosts order is corrupted; the player must repair local name resolution
(all on localhost — no real network).

## Context

Teaches: the resolution chain (`nsswitch.conf` → `files` → hosts), `getent hosts` as the truth
probe vs ping-less debugging, hosts-file hygiene. Single-container per ADR-005; `network: none`
throughout. DESIGN §14 Floor 4 row 1.

## Scope

`scenarios/dns-demon/` only.

## Detailed Requirements

1. `scenario.yaml`: id `dns-demon`, floor 4, difficulty 3, topics `[dns, network, config]`,
   time_estimate_min 25. Install `curl`, `netcat-openbsd`, `libc-bin` tools (getent present in base). Locks:
   - `name-resolves` / `vault.internal answers to its true name` — `getent hosts vault.internal`
     yields `127.0.0.1` (exact match; MSG hints at getent).
   - `courier-delivers` / `The courier completes a delivery` — `curl -fsS http://vault.internal:9000/health`
     returns `VAULT_OK` (retries; app binds 127.0.0.1:9000).
   - `no-lingering-curse` / `No cursed entries remain` — `/etc/hosts` contains no entry mapping
     `vault.internal` to anything other than 127.0.0.1 and no duplicate conflicting lines.
2. Dockerfile + boot:
   - `vaultd` (nc-loop HTTP-ish on 127.0.0.1:9000 answering `VAULT_OK` to /health) started by boot.
   - `courier` loop: every 10s curls the URL, logs failure `courier: cannot resolve vault.internal`
     or connection errors to `/var/log/courier.log`.
   - Breakage layer: `/etc/hosts` gains `203.0.113.66  vault.internal   # migration 2024?` AND a
     second stale line `10.99.0.1 vault.internal`; `/etc/nsswitch.conf` hosts line corrupted to
     `hosts: dns files` (dns first — with network none, dns lookups hang/fail slowly: teaches order
     matters; ensure resolv.conf points nowhere so failure is fast-ish: write `nameserver 127.0.0.99`).
   - Boot-ok, sleep infinity.
3. Fix: restore `hosts: files dns` (or `files` alone) + correct hosts entry `127.0.0.1 vault.internal`
   removing cursed lines.
4. Hints: 01 `curl -v` / courier log — is this a connection problem or a *name* problem? probe with
   `getent hosts vault.internal`. 02 two config files govern local names: `/etc/hosts` content and
   `/etc/nsswitch.conf` order — inspect both. 03 near-answer: fix nsswitch hosts order to
   `files dns`, make hosts map vault.internal → 127.0.0.1 (remove both cursed lines).
5. `solution.md` (+ Lesson: resolution chain, getent as ground truth, hosts-file drift; "dead
   ends" note: why ping is absent — NET_RAW dropped, COOKBOOK §5). `solution.sh`: sed/replace both
   files, verify getent + curl loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed.
- [ ] Fixing hosts but not nsswitch leaves resolution broken/slow → locks catch it (transcript).
- [ ] Room fully functional offline (network none) — no real DNS egress attempted by checks.
- [ ] `ValidateDir` clean; image ≤ 200 MB; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=dns-demon`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Real DNS servers (ADR-005/v2 `internal` networks), resolvectl/systemd-resolved.

## Design References

DESIGN §14 Floor 4; ADR-005; COOKBOOK §4/§5.
