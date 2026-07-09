# Title

Scenario: secrets-specter (Floor 5 — secret rotation drift)

## Summary

Floor-5 room: the vault app can't authenticate to its database because a secret rotation was
half-finished — the DB knows the new password, the app still reads the old one from one of its
THREE config sources; the player must complete the rotation hygienically.

## Context

Teaches: secret-source precedence (env file vs config file vs fallback default), rotation
runbooks, secret file permissions, never-log-secrets discipline. DESIGN §14 Floor 5 row 4;
difficulty 4. Uses PostgreSQL as the credential target (pattern from 51).

## Scope

`scenarios/secrets-specter/` only.

## Detailed Requirements

1. `scenario.yaml`: id `secrets-specter`, floor 5, difficulty 4, topics `[secrets, config, postgres]`,
   time_estimate_min 35; resources `{memory_mb: 1024}`. Locks:
   - `vault-opens` / `The vault app communes again` — app probe `/usr/local/bin/vault-check`
     (connects using the app's real config-resolution order) exits 0 (retries).
   - `old-key-buried` / `The old key is buried` — old password string (`specter-old-2024`) appears
     in NO file under `/etc/vaultapp/` and not in `/root/.pgpass` (grep -r; MSG: it lingers somewhere…).
   - `key-kept-secret` / `The new key is kept like a secret` — `/etc/vaultapp/secret` exists,
     mode 0600, owner root:vaultapp group-readable? — exact: mode 0640 root:vaultapp, and
     `/var/log/vaultapp.log` does NOT contain the new password (`specter-new-2026`) — the
     no-logging discipline check.
2. Dockerfile + boot:
   - Postgres with user `vaultapp`, password ALREADY ROTATED to `specter-new-2026` (DB side done).
   - App config resolution (documented in `/etc/vaultapp/README`): 1) `/etc/vaultapp/secret` file
     if present; 2) `PGPASSWORD` from `/etc/vaultapp/env`; 3) legacy `/etc/vaultapp/config.ini`.
   - Breakage: `secret` file ABSENT; `env` has old password; `config.ini` has old password;
     `/root/.pgpass` old too (four lingering copies incl. .pgpass); a sloppy debug line in
     `vaultapp.log` shows the OLD password (scenery reinforcing the lesson — old only, new never logged by the app).
   - `vaultd` loop + `vault-check` probe + boot-ok + sleep infinity. The NEW password is
     discoverable in-room: the rotation runbook `/root/rotation-runbook.md` states it (simulates
     the operator knowing the new credential) — room stays self-contained.
3. Fix: create `/etc/vaultapp/secret` with new password, 0640 root:vaultapp; purge old password
   from env/config.ini/.pgpass (delete lines or files); do NOT echo the secret into logs
   (the check greps the log — using `echo … | tee /var/log/vaultapp.log` style sloppiness fails it).
4. Hints: 01 the app README explains where it looks for its key, in ORDER — walk the chain and
   compare with the runbook in /root. 02 rotation is only done when the old key exists NOWHERE:
   `grep -r specter-old /etc/vaultapp /root/.pgpass`. 03 near-answer: exact file creation +
   chmod/chown + the three purges.
5. `solution.md` (+ Lesson: precedence chains, half-rotations, secret file perms, grep-audits,
   why secrets never belong in logs). `solution.sh`: full hygienic rotation, verify probes.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: locks 1–2 closed (old key everywhere, app failing), lock 3 closed (secret file absent).
- [ ] Quick-fix route (put new password into `env` only) opens `vault-opens` but leaves `old-key-buried` closed (config.ini/.pgpass remain) — transcript.
- [ ] Sloppy route (secret echoed into the log during fixing) closes `key-kept-secret` — transcript demonstrates the discipline check.
- [ ] `ValidateDir` clean; image ≤ 700 MB (KU-5 table updated); no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=secrets-specter`; transcripts in PR.

## Dependencies

12, 29 (pattern-sibling: 51).

## Non-goals

Real secret managers (mentioned in Lesson as the endgame), TLS client certs, key rotation automation.

## Design References

DESIGN §14 Floor 5; COOKBOOK §4, §8; global secret-handling posture (secrets never generated/handled by agents — mirrored as a game lesson).
