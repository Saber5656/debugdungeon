# Title

Scenario: proxy-maze (Floor 4 — poisoned proxy environment)

## Summary

Floor-4 room: every HTTP tool in the room is shoved through a phantom proxy by layered
`http_proxy`/`HTTPS_PROXY`/`no_proxy` settings scattered across environment files; the player
must trace where each poisoned variable comes from and cleanse them all.

## Context

Teaches: proxy env var semantics (case variants, no_proxy), the env-source maze
(`/etc/environment`, profile.d, bashrc, wrapper scripts), `env | grep -i proxy` discipline.
DESIGN §14 Floor 4 row 4.

## Scope

`scenarios/proxy-maze/` only.

## Detailed Requirements

1. `scenario.yaml`: id `proxy-maze`, floor 4, difficulty 3, topics `[env, network, shell]`,
   time_estimate_min 25. Install `curl`. Locks:
   - `oracle-answers` / `The oracle answers directly` — fresh **login shell** runs
     `curl -fsS --max-time 5 http://127.0.0.1:7000/ask` → `ORACLE_OK`
     (`su -l root -c …` in check; proxied attempt fails fast: phantom proxy points at 127.0.0.1:1
     connection-refused).
   - `no-phantom-guides` / `No phantom guides your steps` — login-shell env contains NO
     `http_proxy|https_proxy|HTTP_PROXY|HTTPS_PROXY|all_proxy|ALL_PROXY` (grep -i on `su -l root -c env`).
   - `courier-cured` / `The courier walks the straight road` — `/usr/local/bin/ask-oracle` wrapper
     works (it had its OWN hardcoded `export https_proxy=…` line — the third hiding spot).
2. Dockerfile breakage (three layers of poison, discovered progressively):
   - `/etc/environment`: `http_proxy=http://127.0.0.1:1` + `HTTPS_PROXY=…` (pam_env — affects login shells).
   - `/etc/profile.d/10-corporate-proxy.sh`: exports lowercase+uppercase pairs + bogus `no_proxy=localhost` **missing 127.0.0.1** (subtle: even after other fixes, tools targeting 127.0.0.1 still proxied — teaching no_proxy nuance; note: many tools treat localhost≠127.0.0.1).
   - `/usr/local/bin/ask-oracle`: wrapper with inline export before its curl call.
   - `oracled` on 127.0.0.1:7000 answering `ORACLE_OK` (nc/sh loop); boot-ok; sleep infinity.
3. Fix: cleanse all three sources (delete lines/files) — or set correct `no_proxy` — locks demand
   *no proxy vars at all* in login env (cleanest lesson) + working wrapper.
4. Hints: 01 curl hangs/refuses — `env | grep -i proxy`; where do these come from? new login shells
   pick them up again. 02 the three classic dens: `/etc/environment`, `/etc/profile.d/*`, and
   inside wrapper scripts themselves — hunt all. 03 near-answer: remove the profile.d file, strip
   /etc/environment lines, sed the wrapper's export line.
5. `solution.md` (+ Lesson: proxy env anatomy, login vs non-login propagation, grep -i env
   discipline, wrapper-script poison; dead end: why unset in current shell didn't persist).
   `solution.sh`: perform all three cleanses, verify via `su -l` probes.

## Acceptance Criteria

- [ ] Harness green both arches; pristine: all three locks closed.
- [ ] Partial cleanses (any 2 of 3 sources) leave ≥1 lock closed with a guiding MSG — matrix table in PR transcript.
- [ ] Checks use `su -l` (fresh login env), never the exec session env — code-reviewed.
- [ ] `ValidateDir` clean; image ≤ 150 MB; no flake ×3.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=proxy-maze`; transcripts in PR.

## Dependencies

12, 29.

## Non-goals

Real proxies, PAC files, apt proxy config (mentioned in solution.md as further dens).

## Design References

DESIGN §14 Floor 4; COOKBOOK §4, §6 (login-shell check nuance shared with 31).
