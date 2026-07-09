# Title

Scenario: postgres-lich (Floor 5 — PostgreSQL won't start / auth misconfig)

## Summary

Floor-5 room: the lich's phylactery (PostgreSQL) is doubly cursed — a bad `postgresql.conf`
parameter prevents startup, and once revived, `pg_hba.conf` rejects the app user; the player must
restore full app→DB connectivity.

## Context

Teaches: reading PG startup logs, `postgresql.conf` hygiene, `pg_hba.conf` auth rules, `pg_isready`,
psql probing. First heavyweight middleware room — KU-5 size measurements happen here.
DESIGN §14 Floor 5 row 1; difficulty 4.

## Scope

`scenarios/postgres-lich/` only.

## Detailed Requirements

1. `scenario.yaml`: id `postgres-lich`, floor 5, difficulty 4, topics `[postgres, database, auth]`,
   time_estimate_min 40; resources `{memory_mb: 1024}`. Install `postgresql` (bookworm's default
   major), `postgresql-client`. **Measure image size on amd64+arm64 and record in PR (KU-5;
   budget 700 MB for Floor 5).** Locks:
   - `phylactery-beats` / `The phylactery accepts connections` — `pg_isready -h 127.0.0.1 -p 5432` (retries ≤ 55s).
   - `app-communes` / `The app user communes with the crypt` —
     `su -s /bin/sh postgres -c ""`-less: run `PGPASSWORD=crypt-pass psql -h 127.0.0.1 -U cryptapp -d cryptdb -tAc 'select 1'` → `1`.
   - `wards-not-wide-open` / `The wards are not flung wide` — `pg_hba.conf` contains no
     `trust` rule for `all`/`0.0.0.0/0` (guards against the chmod-777-of-auth fix; grep-based).
2. Dockerfile + boot:
   - initdb at build (as postgres user); create db `cryptdb`, user `cryptapp` with password
     `crypt-pass`, one table `souls(id int)` with a row (build-time temporary server start — standard pattern).
   - Breakage layer: append `shared_buffers = 128GB` (unstartable on the room's memory) to
     `postgresql.conf`; replace the `host … cryptapp …` hba rule with
     `host cryptdb cryptapp 127.0.0.1/32 reject`.
   - Boot: respawn-loop `pg_ctl start` as postgres (fails → log breadcrumb at
     `/var/log/postgresql/startup.log`), boot-ok regardless, sleep infinity.
   - Listen on 127.0.0.1 only (default). Password auth = scram (bookworm default).
3. Fix path: startup log → fix shared_buffers (sane value or remove line) → server starts →
   psql as cryptapp fails with hba error → fix rule to `scram-sha-256` → `pg_ctl reload`.
4. Hints: 01 the phylactery won't even wake: find postgres's own words
   (`/var/log/postgresql/…`, or run `pg_ctl start` by hand as postgres) — one config value is
   physically impossible. 02 alive but the app is refused: HBA rules are read top-down —
   find the `reject` line; reload, don't restart. 03 near-answer: sed the two lines +
   `su -s /bin/sh postgres -c 'pg_ctl reload -D …'`.
5. `solution.md` (+ Lesson: config-value sanity vs available RAM, hba evaluation order,
   reload vs restart, why `trust` is the forbidden shortcut). `solution.sh`: both seds
   (paths per bookworm layout `/etc/postgresql/<ver>/main/…`), start/reload as postgres, psql verify loop.

## Acceptance Criteria

- [ ] Harness green on amd64 AND arm64 (postgres apt install works both — evidence).
- [ ] Pristine: locks 1–2 closed, `wards-not-wide-open` OPEN (reject≠trust) — ADR-006 ≥1-closed satisfied; noted in PR.
- [ ] `trust`-shortcut route opens lock 2 but closes lock 3 with its MSG — trap transcript.
- [ ] Image size measured & recorded; ≤ 700 MB uncompressed or escalation filed (KU-5).
- [ ] Boot (with failing PG) reaches boot-ok < 20s; solution completes < 120s; no flake ×3.
- [ ] `ValidateDir` clean.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=postgres-lich`; transcripts + size table in PR.

## Dependencies

12, 29.

## Non-goals

Replication, vacuum/bloat lessons, data recovery (52/53 cover data-layer variants).

## Design References

DESIGN §14 Floor 5; KU-5; COOKBOOK §3–§4, §11.
