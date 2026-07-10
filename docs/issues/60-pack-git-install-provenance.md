# Title

Pack install from git URL, pack management (list/remove), provenance completion

## Summary

Extend pack support with git-URL installs (pinned to a commit), `pack list`, `pack remove`, and
complete provenance records for every source kind.

## Context

Git is how community packs will actually circulate. The security posture: we record exactly what
was installed from where (TB4 provenance), and we never auto-update.

## Scope

- `internal/pack/git.go`, `internal/cli/pack_list.go`, `internal/cli/pack_remove.go` + tests
- Not: trust-gate UX (61), registries/auto-update (never in v1 — ADR-007 boundary: git fetch happens via system git, user-initiated)

## Detailed Requirements

1. **Git install**: `pack install <git-url> [--ref <tag|branch|sha>]`:
   - Shell out to **system git** (document requirement git ≥ 2.30; `exec.LookPath` pre-flight,
     friendly error if absent). Ref resolution algorithm (exact):
     - no `--ref` → `git clone --depth 1 -- <url> <tmp>` (default branch);
     - branch/tag ref → `git clone --depth 1 --branch <ref> -- <url> <tmp>`;
     - 40-hex full SHA → `git init <tmp> && git remote add origin <url> && git fetch --depth 1
       origin <sha> && git checkout FETCH_HEAD`;
     - short SHA (7–39 hex) → REJECTED with "use a full commit SHA, branch, or tag" (shallow
       remotes can't resolve abbreviations reliably);
     - unreachable/unknown ref → git's error surfaced sanitized, exit 1.
     Always record `git rev-parse HEAD` as the provenance commit.
   - URL allowlist (exact regexes, table-tested incl. injection attempts `--upload-pack=…`,
     `ext::…`, `file://…`):
     `^https://[A-Za-z0-9.-]+(:[0-9]+)?/[A-Za-z0-9._/-]+(\.git)?$` and scp-form
     `^[A-Za-z0-9_.-]+@[A-Za-z0-9.-]+:[A-Za-z0-9._/-]+(\.git)?$`; everything else rejected.
     Arg-injection hygiene: always `--` separation, URL never interpreted as a flag,
     `-c protocol.ext.allow=never -c protocol.file.allow=never` on every invocation.
   - The cloned tree (minus `.git`) then flows through the EXACT pipeline of 59 (extraction rules
     become tree rules: symlink scan, quotas, validation, disclosure, typed-name confirm).
   - Provenance: `source: {kind:"git", ref:<url>, commit:<sha>, requested_ref:<ref|null>}`.
2. **`pack list`**: table — name, version, scenarios count, source kind + short ref
   (`github.com/x/y@ab12cd3` / path / tarball sha prefix), installed_at, status (`ok` / `broken:
   <reason>` for load-failing packs, from 10's skip-warn path), trust state (`accepted` /
   `not yet played` from provenance.accepted_at — 61 fills semantics).
3. **`pack remove <name>`**: confirm (y/N; `--yes`); refuse while the active run references a
   scenario from it (run store, 20) — message names the run. Removal semantics (exact): rename
   pack dir to sibling `.trash-<ts>` (atomic — the pack is GONE from the registry's view at this
   instant), then best-effort RemoveAll; "no partial state" means no partially-deleted
   `packs/<name>/` is ever observable — a surviving `.trash-*` after a crash is expected and swept
   at next CLI start (shared sweep with 59's --force). Managed images for its scenarios are NOT
   auto-removed (mention `clean --all`).
4. All subcommands work with Docker down.
5. `git` invocations: 60s timeout, output captured to debug log, env scrubbed —
   `GIT_TERMINAL_PROMPT=0`, `GIT_ASKPASS=/bin/true`, AND
   `GIT_SSH_COMMAND=ssh -oBatchMode=yes -oStrictHostKeyChecking=accept-new` (BatchMode is what
   actually stops ssh passphrase/interactive prompts — review catch); private/authed repos
   therefore fail fast with guidance.

## Acceptance Criteria

- [ ] Unit tests: URL regex table (accepts/rejects incl. `--upload-pack=…`, `ext::`, `file://`, short-SHA rejection); git arg construction golden.
- [ ] itest end-to-end git install: a local bare repo fixture served over a REAL allowed transport in-process (`git daemon --export-all` on 127.0.0.1 with a `git://`… NO — keep allowlist intact: serve via `git http-backend` behind `httptest` (stdlib CGI handler) and install from `https://127.0.0.1:<port>/pack.git` with TLS via httptest's cert injected ONLY under the `testhooks` build tag as an extra root CA for the git process (`GIT_SSL_CAINFO` env set by the test). Exercises clone, commit pinning, provenance golden. If the http-backend route proves brittle at implementation, the documented fallback is a plain `http://127.0.0.1` allowance compiled ONLY under `testhooks` — never in release binaries (36's snapshot smoke asserts the flagless binary rejects it).
- [ ] `pack list` golden (ok + broken + differing sources); `pack remove` refusal-while-active covered; crash between trash-rename and RemoveAll (injected) → next CLI start sweeps `.trash-*` (test asserts).
- [ ] Credential prompts provably disabled (env asserted in tests).
- [ ] Provenance golden for git kind (commit sha recorded).

## Validation

`make test` + itest; transcript: git install (public repo or local fixture) → list → play → remove.

## Dependencies

20 (active-run check), 59.

## Non-goals

Pack updates (`remove` + `install` again; v2 may diff), submodules (rejected implicitly by tree scan), auth'd private repos.

## Design References

DESIGN §10.6; ADR-007 (user-initiated fetch only); issue 59 (pipeline reuse).
