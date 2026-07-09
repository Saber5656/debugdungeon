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
     friendly error if absent): `git clone --depth 1 [--branch <ref>] <url> <tmp>` then
     `git rev-parse HEAD`; a full-sha `--ref` uses fetch-by-sha fallback (`git fetch origin <sha> && checkout`).
   - URL allowlist: `https://` and `git@` SSH forms only; reject `file://`, `ext::`, other exotic
     transports (arg-injection hygiene: always `--` separation, URL never interpreted as a flag; validate with a regex + `-c protocol.ext.allow=never`).
   - The cloned tree (minus `.git`) then flows through the EXACT pipeline of 59 (extraction rules
     become tree rules: symlink scan, quotas, validation, disclosure, typed-name confirm).
   - Provenance: `source: {kind:"git", ref:<url>, commit:<sha>, requested_ref:<ref|null>}`.
2. **`pack list`**: table — name, version, scenarios count, source kind + short ref
   (`github.com/x/y@ab12cd3` / path / tarball sha prefix), installed_at, status (`ok` / `broken:
   <reason>` for load-failing packs, from 10's skip-warn path), trust state (`accepted` /
   `not yet played` from provenance.accepted_at — 61 fills semantics).
3. **`pack remove <name>`**: confirm (y/N; `--yes`); refuse while the active run references a
   scenario from it (run store check) — message names the run; removal deletes the pack dir
   atomically (rename to `.trash-<ts>` sibling then RemoveAll — no partial states); managed images
   for its scenarios are NOT auto-removed (mention `clean --all`).
4. All subcommands work with Docker down.
5. `git` invocations: 60s timeout, output captured to debug log, env scrubbed
   (`GIT_ASKPASS=/bin/true`, `GIT_TERMINAL_PROMPT=0` — never prompt for creds; private repos fail fast with guidance).

## Acceptance Criteria

- [ ] itest (local `file://`-free: use a local **bare repo served via `git daemon`? too heavy** → use a plain local path clone through the git kind by allowing `--allow-local-path-git` test-only flag, or construct an https-like fixture via `git init` + direct-dir install for the pipeline and unit-test the URL/arg handling separately) — REQUIRED minimum: unit tests for URL validation/arg construction (table incl. injection attempts `--upload-pack=…`, `ext::`, `file://`) + one end-to-end install from a local git repo via the test-only path, exercising commit pinning + provenance.
- [ ] `pack list` golden (ok + broken + differing sources); `pack remove` refusal-while-active covered; trash-then-delete leaves no partial dir on injected failure.
- [ ] Credential prompts provably disabled (env asserted in tests).
- [ ] Provenance golden for git kind (commit sha recorded).

## Validation

`make test` + itest; transcript: git install (public repo or local fixture) → list → play → remove.

## Dependencies

59 (20 for active-run check).

## Non-goals

Pack updates (`remove` + `install` again; v2 may diff), submodules (rejected implicitly by tree scan), auth'd private repos.

## Design References

DESIGN §10.6; ADR-007 (user-initiated fetch only); issue 59 (pipeline reuse).
