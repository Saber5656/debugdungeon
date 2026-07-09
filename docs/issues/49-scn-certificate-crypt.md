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

1. `scenario.yaml`: id `certificate-crypt`, floor 4, difficulty 4, topics `[tls, openssl, web]`,
   time_estimate_min 35. Install `openssl`, `curl`, and a tiny TLS server — use `openssl s_server`
   itself (present, zero extra deps) wrapped by `seald` loop. Locks:
   - `cert-valid` / `The seal bears a living date` — `openssl x509 -checkend 2592000 -in /etc/seal/seal.crt`
     (≥30 days validity) AND subject CN == `crypt.local` AND issuer == the room CA (MSG per failure).
   - `key-matches` / `Seal and key are one` — cert/key modulus (or pubkey) match; key mode 0600 (MSG for perms too).
   - `crypt-answers` / `The crypt answers over TLS` — `curl -fsS --cacert /etc/seal/ca.crt --resolve crypt.local:8443:127.0.0.1 https://crypt.local:8443/` returns `CRYPT_OK` (retries).
2. Dockerfile (build-time crypto — deterministic enough):
   - Generate room CA (`ca.crt`/`ca.key`, CN=Crypt Root, 10y) at `/etc/seal/`.
   - Generate the EXPIRED server cert: `openssl req … -days 1` then... an already-expired cert is
     awkward at build (can't backdate with plain openssl req) → use `openssl ca`-less trick:
     `-not_before/-not_after` isn't in LibreSSL/openssl req... **Implementation directive**: use
     `faketime`-free approach — `openssl x509 -req` ignores dates before 3.x? Modern OpenSSL 3.x
     (bookworm) supports `openssl x509 -req … -not_before 20240101000000Z -not_after 20240201000000Z`?
     Verify at implementation; **fallback (guaranteed)**: generate with `-days 1` and have the boot
     script `touch` nothing — instead bake a PREGENERATED expired cert+key as literal PEM files in
     `image/` (generated once by the content author, committed; expiry safely in the past, e.g.
     2024). CN mismatch bonus fault: CN=`crypt.locaI` (capital-I homoglyph) on the expired cert —
     subtle read carefully moment.
   - `seald`: loop serving `CRYPT_OK` via `openssl s_server -quiet -accept 8443 -cert seal.crt -key seal.key -www`-style
     (exact serving mode implementer's choice; must return body containing CRYPT_OK over TLS).
   - `/etc/hosts` maps `crypt.local` → 127.0.0.1 (pre-set, not part of the puzzle). Boot-ok; sleep infinity.
3. Fix path: inspect (`openssl x509 -noout -dates -subject`), then issue a new cert signed by the
   room CA (`openssl req -new` + `openssl x509 -req -CA … -days 365 -extfile` SAN=DNS:crypt.local),
   correct CN, key perms 0600, restart seald (respawn loop picks up files each cycle — design seald
   to re-exec per connection so no restart needed; simpler and lock-friendly).
4. Hints: 01 what exactly does the client object to? `curl -v --cacert …` and read the verify
   error; inspect the cert's dates and subject. 02 there's a CA in `/etc/seal` — you can mint a
   fresh cert; mind the CN (look VERY closely at the old one) and SAN. 03 near-answer command pair
   (req + x509 -req with -CA/-CAkey/-set_serial/-days 365 + SAN extfile), chmod 600 key.
5. `solution.md` (+ Lesson: cert lifetimes, CN/SAN, homoglyph vigilance, local CAs; dead-ends:
   why not just `-k` — the lock demands verification). `solution.sh`: full openssl command
   sequence, curl verify loop.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed (expired + homoglyph CN + old key perms 0644).
- [ ] `curl -k` route does NOT open `crypt-answers` (lock uses --cacert verification) — verified.
- [ ] SAN present in solution cert (modern curl requires SAN, not CN — check enforces via successful verify).
- [ ] `ValidateDir` clean; image ≤ 200 MB; no flake ×3; pregenerated-PEM fallback documented if used.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=certificate-crypt`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Public CAs/ACME, nginx TLS termination, mTLS (v2 room idea).

## Design References

DESIGN §14 Floor 4; COOKBOOK §4, §10 (arch-neutral: openssl via apt).
