# Title

Scenario: certificate-crypt (Floor 4 — expired local TLS)

## Summary

Floor-4 room: the crypt's seal (a local HTTPS service) rejects all visitors because its
self-signed certificate expired; the player must inspect the cert, mint a valid one with the
in-room CA key, and restore trusted local HTTPS.

## Context

Teaches: `openssl x509 -dates/-subject`, verify errors, issuing certs from a local CA, cert/key
file hygiene, curl `--cacert`. Fully offline (self-contained CA). DESIGN §14 Floor 4 row 3;
difficulty 4.

## Scope

`scenarios/certificate-crypt/` only.

## Detailed Requirements

1. `scenario.yaml` (C3): id `certificate-crypt`, floor 4, difficulty 4, topics `[tls, openssl, web]`,
   time_estimate_min 35. TLS server = `openssl s_server` wrapped by the `seald` loop (zero extra deps). Locks:
   - `cert-valid` / `The seal bears a living date` — exact commands:
     `openssl x509 -checkend 2592000 -noout -in /etc/seal/seal.crt` (≥30 days) AND
     `openssl x509 -noout -subject -nameopt RFC2253 -in …` contains `CN=crypt.local` (byte-exact,
     homoglyph fails) AND `openssl x509 -noout -issuer -nameopt RFC2253 -in …` contains
     `CN=Crypt Root CA`. Timeout 10; MSG names the first failing clause.
   - `key-matches` / `Seal and key are one` — pubkey compare:
     `openssl x509 -noout -pubkey -in seal.crt | sha256sum` equals
     `openssl pkey -pubout -in seal.key | sha256sum` AND key mode is exactly 0600
     (`stat -c %a` == `600`). Timeout 10; MSG distinguishes mismatch vs perms.
   - `crypt-answers` / `The crypt answers over TLS` —
     `curl -fsS --cacert /etc/seal/ca.crt --resolve crypt.local:8443:127.0.0.1 https://crypt.local:8443/`
     body contains `CRYPT_OK` (full verification — no `-k`; retry ≤ 25s, timeout 30).
2. Cert model (STATED PRECISELY — the review caught self-signed/CA ambiguity): the room has a
   self-signed root CA (`/etc/seal/ca.crt|ca.key`, CN=`Crypt Root CA`, 10y validity, generated at
   BUILD time — long-lived, safe to generate) which signs the SERVER cert. The pristine server
   cert is **CA-signed but expired, with the homoglyph CN** — clients trusting ca.crt still refuse
   it (expiry + name), which is the puzzle.
3. Dockerfile (C3 preamble; apt: `openssl`, `curl`):
   - Expired server cert+key: **committed PEM fixtures** `image/seal.expired.crt` /
     `image/seal.expired.key` (single mandated path — no build-time backdating tricks). Generated
     once by the content author from the room CA with validity 2024-01-01 → 2024-02-01,
     CN=`crypt.locaI` (capital-I homoglyph), SAN absent (period-typical sloppiness); COPY'd to
     `/etc/seal/seal.crt` (0644) and `/etc/seal/seal.key` (**0644 — deliberately sloppy**, closing
     `key-matches`' perm clause on pristine). The author commits the generation commands in
     `image/FIXTURES.md` for future regeneration.
   - `seald` (`/usr/local/bin/seald`, exact loop — re-reads cert files every connection):
     ```sh
     #!/bin/sh
     while true; do
       openssl s_server -accept 8443 -naccept 1 -quiet \
         -cert /etc/seal/seal.crt -key /etc/seal/seal.key < /etc/seal/response.http \
         >>/var/log/seald.log 2>&1 || sleep 1
     done
     ```
     where `/etc/seal/response.http` holds raw HTTP bytes with body `CRYPT_OK` (s_server streams
     stdin to the client). Verified at implementation; if `-naccept 1 -quiet` streaming misbehaves
     on bookworm's OpenSSL, the documented alternative is `-www` + lock matching on the status
     page containing `CRYPT_OK` marker text — pick ONE and freeze it in the committed room.
   - `/etc/hosts` alias: Docker manages `/etc/hosts` at create time (image edits are LOST — review
     catch) → the BOOT script appends `127.0.0.1 crypt.local` idempotently
     (`grep -q crypt.local /etc/hosts || echo '127.0.0.1 crypt.local' >> /etc/hosts`); writable
     under the profile (it's a bind-managed file, root can append). Boot: start seald loop, hosts
     line, C3 boot-ok; sleep infinity.
4. Fix path: inspect (`openssl x509 -noout -dates -subject`), then issue a new cert signed by the
   room CA (`openssl req -new` + `openssl x509 -req -CA … -days 365 -extfile` SAN=DNS:crypt.local),
   correct CN, key perms 0600 — no service restart needed (seald re-execs per connection).
5. Hints: 01 what exactly does the client object to? `curl -v --cacert …` and read the verify
   error; inspect the cert's dates and subject. 02 there's a CA in `/etc/seal` — you can mint a
   fresh cert; mind the CN (look VERY closely at the old one) and SAN. 03 near-answer command pair
   (req + x509 -req with -CA/-CAkey/-set_serial/-days 365 + SAN extfile), chmod 600 key.
6. `solution.md` (+ Lesson: cert lifetimes, CN/SAN, homoglyph vigilance, local CAs; dead-ends:
   why not just `-k` — the lock demands verification). `solution.sh`: full openssl command
   sequence, curl verify loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed (expired+homoglyph cert, key mode 0644, curl refusal).
- [ ] `curl -k` route does NOT open `crypt-answers` (lock uses --cacert verification) — verified.
- [ ] SAN present in solution cert (modern curl requires SAN, not CN — check enforces via successful verify).
- [ ] `ValidateDir` clean; image ≤ 200 MB; no flake ×3; `image/FIXTURES.md` regeneration commands committed; seald serving mode frozen (streamed response.http or -www variant, one chosen).

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=certificate-crypt`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Public CAs/ACME, nginx TLS termination, mTLS (v2 room idea).

## Design References

DESIGN §14 Floor 4; COOKBOOK §4, §10 (arch-neutral: openssl via apt).
