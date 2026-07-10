# Title

Scenario spec types and strict YAML loader

## Summary

Implement the `scenario.yaml` v1 data model (`internal/scenario`): Go types mirroring DESIGN §6.2,
strict YAML decoding (unknown fields rejected), and default application. Validation semantics live
in issue 08.

## Context

The spec is the contract between engine and content (bundled and community). Strictness at the
parse layer is a security requirement (DESIGN §10.4: no silent capability creep).

## Scope

- `internal/scenario/spec.go`, `internal/scenario/load.go` + table-driven tests
- Not: semantic validation (08), hashing (09), registry (10)

## Detailed Requirements

1. Types (yaml tags exactly as in DESIGN §6.2):
   ```go
   type Spec struct {
     SchemaVersion int      `yaml:"schema_version"`
     ID            string   `yaml:"id"`
     Title         string   `yaml:"title"`
     Floor         int      `yaml:"floor"`
     Difficulty    int      `yaml:"difficulty"`
     Topics        []string `yaml:"topics"`
     TimeEstimateMin int    `yaml:"time_estimate_min"`
     Lore          string   `yaml:"lore"`
     Entry         Entry    `yaml:"entry"`
     Build         Build    `yaml:"build"`
     Locks         []Lock   `yaml:"locks"`
     Hints         []Hint   `yaml:"hints"`
     Solution      Solution `yaml:"solution"`
     Resources     Resources `yaml:"resources"`
     Mounts        Mounts   `yaml:"mounts"`
     Network       string   `yaml:"network"`
   }
   type Entry struct{ User, Shell, Workdir string }
   type Build struct{ Context string }
   type Lock struct{ ID, Name, Script string; TimeoutSec int `yaml:"timeout_sec"` }
   type Hint struct{ File string }
   type Solution struct{ Walkthrough string `yaml:"walkthrough"`; Script string `yaml:"script"` }
   type Resources struct{ MemoryMB int `yaml:"memory_mb"`; CPUs float64 `yaml:"cpus"`; Pids int `yaml:"pids"` }
   type Mounts struct{ Tmpfs []TmpfsMount `yaml:"tmpfs"` }
   type TmpfsMount struct{ Path string; SizeMB int `yaml:"size_mb"`; NrInodes int `yaml:"nr_inodes"` }
   ```
2. `LoadSpec(fsys fs.FS, dir string) (*Spec, error)` reads `path.Join(dir, "scenario.yaml")`
   (`dir` must satisfy `fs.ValidPath` — reject absolute or `..`; ≤ 64 KiB cap, larger → error
   whose text contains `scenario.yaml` and `64 KiB`) and decodes with `yaml.v3`
   `Decoder.KnownFields(true)`. Additionally, a pre-decode `yaml.Node` walk rejects duplicate
   mapping keys at every nesting level (KnownFields alone doesn't) with key name + line.
   Error contract: path always present; yaml line for parse/decode/duplicate errors; size/read
   errors carry path + reason only.
3. **Omitted-vs-zero handling**: decode into an internal raw struct using pointer fields for every
   defaultable scalar (`*int`, `*float64`, `*string`), then convert to the value-typed `Spec`,
   defaulting only nil pointers: `entry.user=root`, `entry.shell=/bin/bash`, `entry.workdir=/root`,
   `build.context=./image`, per-lock `timeout_sec=10`,
   `resources={memory_mb:512, cpus:1.0, pids:256}`, `network=none`. Explicit zeros survive into
   `Spec` so the validator (08) rejects them (`timeout_sec: 0` must NOT silently become 10).
4. No filesystem access beyond `scenario.yaml` in this issue. No validation beyond decode.
5. Errors are wrapped `fmt.Errorf("scenario %s: ...", dir, ...)` style; map to exit code 4 at CLI layer later.

## Acceptance Criteria

- [ ] Golden test: a fully-populated manifest round-trips into the expected struct.
- [ ] Defaults test: minimal manifest yields the documented defaults.
- [ ] Rejection tests: unknown top-level field, unknown nested field (`entry.uzer`), wrong type (`floor: "one"`), duplicate key (top-level AND nested), file > 64 KiB — each yields a distinct error containing the offending name (oversize: contains `scenario.yaml` + `64 KiB`).
- [ ] Explicit-zero test: `timeout_sec: 0` and `memory_mb: 0` survive as 0 in `Spec` (not defaulted).
- [ ] `LoadSpec` works against both `embed.FS` and `os.DirFS` (test both).

## Validation

`go test ./internal/scenario/...` green; include a `testdata/spec/` corpus with ≥ 8 manifests
(valid + each rejection case).

## Dependencies

01.

## Non-goals

Semantic/security validation (08), JSON Schema artifact (08), registry/embed (10).

## Design References

DESIGN §6.2, §10.4 (TB3 strict decode).
