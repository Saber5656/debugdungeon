# Title

Scenario: nginx-gargoyle (Floor 4 — broken reverse proxy)

## Summary

Floor-4 room: the stone gargoyle (nginx) guards the gallery but serves only errors — a config
syntax fault plus an upstream port mismatch to the local backend; the player must lint, fix, and
reload nginx properly.

## Context

Teaches: `nginx -t` as the config linter, error vs access logs, upstream/proxy_pass anatomy,
reload vs restart. First real middleware room (apt: nginx). DESIGN §14 Floor 4 row 2.

## Scope

`scenarios/nginx-gargoyle/` only.

## Detailed Requirements

1. `scenario.yaml`: id `nginx-gargoyle`, floor 4, difficulty 3, topics `[nginx, web, config]`,
   time_estimate_min 30. Install `nginx`, `curl` (--no-install-recommends; verify size budget). Locks:
   - `config-sound` / `The gargoyle's runes parse` — `nginx -t` exits 0 (MSG: run it yourself).
   - `gargoyle-awake` / `The gargoyle stands watch` — master process running (retries).
   - `gallery-served` / `The gallery is open to visitors` — TWO conditions (proxy-proof, review
     catch): `curl -fsS http://127.0.0.1:80/` body contains `GALLERY_OK` AND the active config
     proxies to the true backend (`nginx -T 2>/dev/null | grep -q 'proxy_pass http://127.0.0.1:9107;'`)
     — a static-file impostor response cannot pass. Retry ≤ 25s, timeout 30.
2. Dockerfile + boot (C3 preamble; apt: `nginx`, `curl`, `netcat-openbsd`):
   - Backend `galleryd`: same reference nc-loop as issue 47's `vaultd` (port **9107**, body `GALLERY_OK`).
   - nginx site config `/etc/nginx/sites-enabled/gallery.conf` — EXACT shipped content (so
     solution.sh seds are stable):
     ```
     server {
         listen 80 default_server
         location / {
             proxy_pass http://127.0.0.1:97107;
         }
     }
     ```
     (two faults: missing `;` after `default_server`, wrong upstream port `97107`.)
   - Remove default site. Boot (C3): start galleryd loop; run `nginx -t >>/var/log/nginx/config-check.log 2>&1 || true`
     (the breadcrumb — nginx's own parser output, captured explicitly since a non-starting nginx
     writes no error.log); boot-ok marker; sleep infinity.
     (nginx not respawned by boot — starting it correctly is the player's job; stated in check MSGs.)
3. Fix: add semicolon, correct port 9107, `nginx -t`, start nginx (`nginx` binary directly;
   `service nginx start` also works if init script present — accept either; locks only test outcomes).
4. Hints: 01 is the gargoyle even alive? (`ps`, `curl -v 127.0.0.1`); nginx configs have a linter:
   `nginx -t` — and someone left its words in `/var/log/nginx/config-check.log`. 02 after the
   syntax fix: 502s — where does proxy_pass point, and where does the backend actually listen
   (`ss -tlnp`)? 03 near-answer: semicolon + 9107 + start nginx.
5. `solution.md` (+ Lesson: config linters first; 502 = upstream mismatch; captured-parser-output
   trick; reload discipline). `solution.sh` (stable against the exact shipped config):
   ```sh
   sed -i 's/listen 80 default_server$/listen 80 default_server;/' /etc/nginx/sites-enabled/gallery.conf
   sed -i 's#proxy_pass http://127.0.0.1:97107;#proxy_pass http://127.0.0.1:9107;#' /etc/nginx/sites-enabled/gallery.conf
   nginx -t && nginx
   ```
   then curl retry loop ≤ 20s.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed.
- [ ] Syntax-only fix → nginx starts but 502 → `gallery-served` closed with useful MSG (scripted transcript: apply first sed only, start, run checks).
- [ ] Image ≤ 250 MB uncompressed (nginx adds bulk — measure, record in PR; KU-5 watch).
- [ ] `ValidateDir` clean; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=nginx-gargoyle`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

TLS at nginx (49 owns TLS), load balancing, systemd unit management.

## Design References

DESIGN §14 Floor 4; COOKBOOK §3–§4 (no-systemd service pattern), §11 (size budget).
