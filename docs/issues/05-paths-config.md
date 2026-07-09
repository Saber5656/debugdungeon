# Title

State paths and configuration resolution

## Summary

Implement `internal/config`: per-OS state/config/log path resolution (DESIGN §8.1), the
`DEBUGDUNGEON_HOME` override, directory creation with strict permissions, and strict-YAML config
file loading with documented precedence.

## Context

Every store (run, progress, packs, logs) builds on these paths. Getting permissions and precedence
wrong is a security/consistency defect, so this lands before any store.

## Scope

- `internal/config/paths.go`, `internal/config/config.go` + tests
- Not: the stores themselves (20, 26), logging sink (06)

## Detailed Requirements

1. `type Paths struct { StateRoot, ConfigFile, ProgressFile, RunFile, RunLockFile, PacksDir, LogDir, LogFile string }`.
2. `ResolvePaths(dataDirFlag string) (Paths, error)` precedence:
   1. `--data-dir` flag value, if non-empty
   2. `DEBUGDUNGEON_HOME` env, if non-empty
   3. Per-OS defaults — exactly DESIGN §8.1:
      - darwin: state root `~/Library/Application Support/debugdungeon`, logs `~/Library/Logs/debugdungeon`, config file inside state root.
      - linux: data `${XDG_DATA_HOME:-~/.local/share}/debugdungeon`, config `${XDG_CONFIG_HOME:-~/.config}/debugdungeon/config.yaml`, logs `${XDG_STATE_HOME:-~/.local/state}/debugdungeon`.
   When an override (1/2) is used, ALL paths (config, data, logs, packs) nest under it.
3. `EnsureDirs(p Paths) error`: create missing dirs `0700`; new files created by stores must use `0600` (document; enforced in 20/26).
4. Config file (`config.yaml`), strict decode (unknown fields = error), all fields optional:
   ```yaml
   free_roam: false        # bool, disables floor gating (DESIGN §3.5)
   no_color: false         # bool
   update_check: false     # bool, reserved; read but unused until issue 66
   ```
   Missing file → zero-value config, not an error. Malformed file → error mentioning the path
   (exit code 1 path; no quarantine for config).
5. Effective settings precedence (per setting): CLI flag > env (`NO_COLOR`) > config file > default.
   Expose `type Config struct` + `Load(paths Paths) (Config, error)` + `Effective(flags…)` helper consumed by 04's flag layer.
6. All functions must be testable with `t.Setenv` + temp dirs; no global state besides a lazy singleton in `cli` wiring.

## Acceptance Criteria

- [ ] Table-driven tests cover: darwin defaults, linux XDG set/unset, `DEBUGDUNGEON_HOME`, `--data-dir` beating env.
- [ ] Dirs created `0700`; test asserts mode bits (skip strict assert on non-POSIX).
- [ ] Unknown config key yields an error naming the key and the file path.
- [ ] `free_roam: true` observable via `Config`.

## Validation

`go test ./internal/config/...` green including permission-mode assertions; run the built binary
with `DEBUGDUNGEON_HOME=$(mktemp -d) debugdungeon version` and show the tree created (nothing yet
except dirs when a command touches them — `version` must NOT create dirs; add test).

## Dependencies

04.

## Non-goals

progress/run schemas (20/26), config editing commands, Windows paths.

## Design References

DESIGN §8.1, §5.2, §10.4; ADR-007 (no network → no remote config).
