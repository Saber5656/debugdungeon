# DebugDungeon — v1 Design Document

Status: Draft for review (2026-07-10)
Owner: Saber5656
Canonical location: `docs/DESIGN.md` (this file is the source of truth for product design; GitHub Issues are derived artifacts)

---

## 1. Product Overview

### 1.1 Pitch

**DebugDungeon** is a debugging escape game for the terminal.
Each "room" of the dungeon is a **real, intentionally broken Linux environment** running in a
Docker container on the player's machine. The player enters the room with a real shell,
diagnoses the breakage with real tools (`ls`, `journalctl`-free log files, `ps`, `ss`, `nginx -t`,
`psql`, …), repairs the environment, and types `escape`. If every **lock** (an automated check)
opens, the player escapes the room and descends deeper into the dungeon.

It is a single-player, local-first, open-source CLI game. There is no server component,
no account, and no telemetry.

### 1.2 Product name and identifiers

| Item | Value | Notes |
|---|---|---|
| Product name | DebugDungeon | Repo: `Saber5656/debugdungeon` |
| CLI binary | `debugdungeon` | `dd` is impossible (coreutils collision) |
| Short alias | `ddgn` | Installed as a symlink next to the binary |
| Go module | `github.com/Saber5656/debugdungeon` | |
| Docker label namespace | `com.debugdungeon.*` | Used to find/GC our containers & images |
| Docker image tag prefix | `debugdungeon/scn-<scenario-id>:<content-hash>` | Local tags only, never pushed |
| License | MIT | Decision by owner, 2026-07-10 |

See ADR-008 for naming rationale.

### 1.3 Target users and difficulty posture

Broad range, explicitly both ends (owner decision, 2026-07-10):

| Persona | Needs | Design response |
|---|---|---|
| P1: Learner (student / junior dev) | Guidance, safety, gradual curve | Tutorial room, progressive hints, floor gating, solution walkthroughs |
| P2: Working developer / SRE | Realistic faults, minimal hand-holding | Real services (nginx, PostgreSQL, Redis), difficulty 3–5 rooms, hints optional and counted |
| P3: Content author (post-MVP) | Create and share rooms | Scenario spec + authoring kit (`scenario init/validate/test`), pack install |

### 1.4 Differentiation (see docs/research/comparative-landscape.md)

- Versus SadServers / hosted labs: fully local, offline after image build, no account, OSS, game framing.
- Versus OverTheWire/CTFs: focuses on *operational debugging* (services, configs, disks), not exploitation.
- Versus code katas (Exercism etc.): the artifact being fixed is an **environment**, not source code.

### 1.5 Explicit product principles

1. **Real over simulated.** Real containers, real tools, real man pages. No fake shell.
2. **The host is sacred.** A scenario must never be able to damage the player's machine. Secure defaults are not scenario-overridable.
3. **One terminal is enough.** The whole game loop works inside a single terminal session.
4. **Content is data.** Scenarios are declarative directories; the engine has no scenario-specific code.
5. **Everything solvable, provably.** Every scenario ships a machine-executable solution that CI replays against the locks (ADR-006).

---

## 2. Goals and Non-Goals

### 2.1 v1 MVP goals (release gate)

- Playable game loop: `list → play → (shell) → hint/reset → escape/check → victory → progress`.
- 4 bundled scenarios (tutorial + Floor 1) with hints and solutions.
- Hardened container security profile (§10) enforced and regression-tested.
- Installable via Homebrew tap and GitHub Releases binaries (macOS arm64/amd64, Linux arm64/amd64).
- Documentation: README quickstart, player guide, security policy.

### 2.2 v1 full goals (all planned issues; post-MVP waves)

- 20 bundled scenarios across 5 floors + capstone (§14).
- Authoring kit: `scenario init | validate | test` + public authoring guide.
- Community scenario packs: install/uninstall from local path, tarball, or git URL, with an explicit trust gate (§10.6).
- Gamification: dungeon map, statistics, achievements, per-floor ambience.

### 2.3 Non-goals for v1 (deliberate)

| Non-goal | Rationale / future |
|---|---|
| Hosted service, accounts, shared leaderboards | No server operation; deferred v2+, would need its own security design |
| Multi-container scenarios (compose topologies) | Engine complexity; single container only (ADR-005). Schema reserves room |
| Browser/WASM simulation mode | Different execution engine; v2 idea |
| Windows native (non-WSL2) support | Docker+PTY complexity; WSL2 is best-effort documented path |
| Anti-cheat | Local single-player; players can inspect images — reading checks is a legitimate "hint" |
| i18n of game content | English content v1; Japanese README translation optional non-blocking |
| systemd inside scenarios | Heavy/fragile in containers; cookbook mandates boot-script pattern (§7.4) |
| Telemetry / analytics / auto-update | No **implicit** network I/O from the CLI (ADR-007); the only network operations are user-initiated (§10.8); opt-in update *check* is a Wave 9 item, off by default |

---

## 3. Player Experience

### 3.1 First-run flow

```
$ brew install saber5656/tap/debugdungeon
$ debugdungeon doctor        # verifies Docker reachability, arch, disk space
$ debugdungeon list          # shows Floor 1 rooms; deeper floors locked
$ debugdungeon play welcome-cell
```

`play` builds (first time) the scenario image, creates the container with the hardened
profile, prints the room's lore banner, and drops the player into a shell **inside** the room.

### 3.2 In-room experience (single-terminal protocol)

Inside the room, three helper commands are injected at `/dungeon/bin` (also on `PATH`):

| In-room command | Effect | Mechanism (§7.5) |
|---|---|---|
| `escape` | Attempt to escape: run all locks | shell exits with code 42; host runs locks |
| `hint` | Reveal the next hint | shell exits with code 43; host prints hint, re-enters room |
| `giveup` | Abandon and reveal the solution | shell exits with code 44; host confirms, reveals |
| (plain `exit` / Ctrl-D) | Leave the dungeon; run is paused, container kept | normal exit code |

Example transcript (abridged):

```
$ debugdungeon play rusty-path
⚔  Floor 1 — Room 2: The Rusty PATH        difficulty ★☆☆☆☆
   Something corrupted the ancient PATH sigils. The gate mechanism
   `open-gate` exists somewhere, but the walls no longer find it…
   Locks: [path-restored] [gate-opened]     Hints: 0/3 used
   Type `escape` to attempt escape, `hint` for a hint.

hero@rusty-path:~$ open-gate
bash: open-gate: command not found
hero@rusty-path:~$ hint
  Hint 1/3: Where does your shell look for commands? Compare `echo $PATH`
  with what login shells are given in /etc/profile…
hero@rusty-path:~$ ... (player fixes /etc/profile, runs open-gate) ...
hero@rusty-path:~$ escape
  Testing lock [path-restored] … OPEN
  Testing lock [gate-opened]   … OPEN
🏆 ESCAPED in 07:42 with 1 hint. Room cleared!
   Next: `debugdungeon play forbidden-scroll`
```

If any lock stays closed, the failed lock names and their (player-safe) failure messages are
printed and the shell re-opens in the same container — state intact.

### 3.3 Session continuity rules

- Exiting the shell normally **pauses** the run: the container is kept (stopped is allowed but v1 keeps it running), and `debugdungeon play <same-id>` re-attaches.
- Only **one active run** at a time in v1. `play <other>` with an active run errs with guidance (`--force` abandons the current run after confirmation).
- `debugdungeon check|hint|status` also work from a second terminal against the active run; identical code paths.
- Re-entry after `hint`/failed `escape` starts a new exec session: working directory resets to the scenario workdir; shell history persists via `HISTFILE=/root/.dungeon_history` (or entry user's home).

### 3.4 Difficulty, hints, scoring

- Difficulty: integer 1–5, displayed as stars. Labels: 1 Novice, 2 Adept, 3 Skilled, 4 Expert, 5 Nightmare.
- Hints: ordered list per scenario (2–4 recommended). Revealing is persisted per run and counted in the clear record. No point penalties in v1 — the *count* is the score companion.
- Clear record per scenario: best wall-clock time (from run start to successful escape; resets do **not** reset the clock), hints used, resets used, clears count.
- `give-up` marks the scenario as `given_up` (revealable solution, replayable; does not count as clear).

### 3.5 Progression: floors and gating

| Floor | Theme | Rooms | Difficulty | Unlock rule |
|---|---|---|---|---|
| F1 The Entrance Halls | shell & filesystem basics | 4 (incl. tutorial) | 1 | always open |
| F2 The Daemon Warrens | processes & services | 4 | 2 | ≥2 clears on F1 |
| F3 The Flooded Archives | disks, files at scale, logs | 3 | 3 | ≥2 clears on F2 |
| F4 The Tangled Battlements | in-container networking & web | 4 | 3–4 | ≥2 clears on F3 |
| F5 The Data Crypts | databases & app data | 4 | 4 | ≥2 clears on F4 |
| Capstone: The Cascade Throne | multi-fault finale | 1 | 5 | ≥12 **distinct** rooms cleared |

`--free-roam` (config `free_roam: true`) disables gating for practice/classroom use.

---

## 4. System Architecture

### 4.1 Component overview

```mermaid
flowchart LR
    subgraph Host["Host (trusted)"]
      CLI[debugdungeon CLI]
      ST[(State dir\nprogress.json / run.json)]
      EMB[(Embedded scenario bundle\n+ installed packs)]
      CLI --> ST
      CLI --> EMB
    end
    DD[Docker daemon] -->|creates| C[Scenario container\n(semi-trusted, hardened)]
    CLI -->|Engine API via socket| DD
    CLI -->|exec: interactive shell| C
    CLI -->|exec: lock scripts, stdin-piped| C
```

### 4.2 Go module map

| Package | Responsibility | Key issue |
|---|---|---|
| `cmd/debugdungeon` | main; wiring only | 04 |
| `internal/cli` | cobra commands, one file per command | 04, 21–28 |
| `internal/config` | config file + env + flags resolution; path layout (§8.1) | 05 |
| `internal/logx` | debug log file + user-facing error rendering | 06 |
| `internal/scenario` | spec types, strict YAML load, validation, content hash, registry (embedded + installed) | 07–10 |
| `internal/textsafe` | sanitization of scenario-sourced text (§10.5) | 11 |
| `internal/dockerx` | Docker client wrapper: ping/info, build, create/start/stop/remove, exec, GC | 13–16 |
| `internal/session` | interactive TTY session + sentinel-exit-code protocol (§7.5) | 17, 18 |
| `internal/locks` | lock runner (§7.6) | 19 |
| `internal/game` | run store + state machine, orchestration of play/check/hint/reset/give-up | 20–25 |
| `internal/progress` | progress store, unlock rules, stats | 26 |
| `internal/ui` | banners, tables, map rendering, styles (lipgloss), NO_COLOR support | 27, 28, 64, 65 |
| `scenarios/` | bundled content, embedded via `go:embed` | 30–33, 40–55 |

Dependency budget (v1): `spf13/cobra`, `github.com/docker/docker` (client), `gopkg.in/yaml.v3`,
`charmbracelet/lipgloss`, `golang.org/x/term`, `golang.org/x/sys`. TUI (`bubbletea`) enters only in Wave 9.
Every new dependency beyond this list requires an ADR note (supply-chain posture, §10.7).

### 4.3 Toolchain

- Go **1.26.x** (latest stable as of 2026-07; `go.mod` `go 1.26`). Minimum supported for contributors: 1.25.
- Docker Engine API ≥ **1.44** negotiated via SDK version negotiation; works with Docker Desktop, OrbStack, colima, Rancher Desktop (moby), and rootless Docker. Podman socket-compat is best-effort, untested in CI (known unknown KU-2).

---

## 5. CLI Command Surface

### 5.1 Commands (full v1 plan; wave in parentheses)

| Command | Purpose | Wave |
|---|---|---|
| `debugdungeon list` | Table of scenarios: floor, difficulty, status (🔒 locked / ⬜ new / ✅ cleared / 🏳 given-up) | 3 |
| `debugdungeon play <scenario-id>` | Start or resume a run, enter the room shell | 3 |
| `debugdungeon check` | Run locks against active run from outside | 3 |
| `debugdungeon hint` | Reveal next hint of active run | 3 |
| `debugdungeon reset` | Recreate active run's container from image (state wiped, clock keeps running) | 3 |
| `debugdungeon give-up [--no-reveal]` | End active run as given-up; reveal solution unless suppressed | 3 |
| `debugdungeon solution <scenario-id>` | Show walkthrough (only if cleared or given-up) | 3 |
| `debugdungeon status` | Active run summary (room, elapsed, hints, locks last result) | 3 |
| `debugdungeon map` | Dungeon map: floors, rooms, clears, gating | 3 (static) / 9 (TUI) |
| `debugdungeon doctor` | Environment diagnosis (Docker, arch, disk, state dir) | 2 |
| `debugdungeon clean [--all]` | Remove paused run container / all `com.debugdungeon` containers+images | 2 |
| `debugdungeon version` | Version, commit, Go version, platform (Docker API version is `doctor`'s job) | 0 |
| `debugdungeon completion <shell>` | cobra-generated completions | 0 |
| `debugdungeon scenario init <dir>` | Scaffold a new scenario | 7 |
| `debugdungeon scenario validate <dir>` | Validate spec + static security rules | 7 |
| `debugdungeon scenario test <dir>` | Build, apply `solution.sh`, assert locks open (solvability) | 7 |
| `debugdungeon pack install <path\|tar\|git-url>` | Install a community scenario pack (trust gate §10.6) | 8 |
| `debugdungeon pack list` / `pack remove <name>` | Manage installed packs | 8 |
| `debugdungeon stats` | Aggregate play statistics | 9 |

### 5.2 Global flags and environment

| Flag / env | Meaning |
|---|---|
| `--verbose` / `-v` | Debug logging to stderr in addition to log file |
| `--no-color` / `NO_COLOR` | Disable ANSI styling |
| `--data-dir` / `DEBUGDUNGEON_HOME` | Override the whole state root (tests, classrooms) |
| `DOCKER_HOST` etc. | Respected via Docker SDK default env handling |

### 5.3 Process exit codes (CLI contract)

| Code | Meaning |
|---|---|
| 0 | Success (for `check`: escaped) |
| 1 | Generic runtime error |
| 2 | Usage error (cobra) |
| 3 | Docker unavailable / incompatible |
| 4 | Scenario not found or invalid |
| 5 | No active run (for run-scoped commands) |
| 6 | Locks failed (`check` command when not all locks open) |
| 7 | Scenario locked by floor gating |
| 10 | Internal bug (panic trap) — asks for bug report |

Reserved in-container sentinel exit codes (never CLI exit codes): 42 escape, 43 hint, 44 giveup (§7.5).

---

## 6. Scenario Format Specification

### 6.1 Directory layout (one scenario = one directory)

```
scenarios/<id>/
├── scenario.yaml          # manifest (required)
├── image/
│   ├── Dockerfile         # environment incl. baked-in breakage (required)
│   └── ...                # files referenced by the Dockerfile
├── checks/
│   └── <lock-id>.sh       # one POSIX sh script per lock (required ≥1)
├── hints/
│   ├── 01.md … NN.md      # progressive hints (required ≥1)
├── solution.md            # human walkthrough (required)
└── solution.sh            # machine-executable solution used by `scenario test` (required)
```

### 6.2 `scenario.yaml` schema (v1 = `schema_version: 1`)

Strict decoding: unknown fields are a **validation error** (prevents silent capability creep).

| Field | Type | Req | Constraints |
|---|---|---|---|
| `schema_version` | int | ✔ | must be `1` |
| `id` | string | ✔ | `^[a-z0-9][a-z0-9-]{2,39}$`; must equal directory name; unique across registry |
| `title` | string | ✔ | ≤ 60 chars after sanitization |
| `floor` | int | ✔ | 1–5, or 6 (capstone) |
| `difficulty` | int | ✔ | 1–5 |
| `topics` | []string | ✔ | 1–6 items, each `^[a-z0-9-]{2,20}$` |
| `time_estimate_min` | int | ✔ | 5–120 |
| `lore` | string | ✔ | ≤ 1500 chars; rendered sanitized (§10.5) |
| `entry.user` | string |  | default `root`; must exist in image |
| `entry.shell` | string |  | default `/bin/bash`; **in schema v1 the only allowed value is `/bin/bash`** (helpers rely on bash login semantics; field exists for future widening) |
| `entry.workdir` | string |  | default `/root`; absolute path |
| `build.context` | string |  | default `./image`; must resolve inside scenario dir, no symlink escape |
| `locks[]` | list | ✔ | 1–8 items |
| `locks[].id` | string | ✔ | `^[a-z0-9-]{2,32}$`, unique in file |
| `locks[].name` | string | ✔ | ≤ 60 chars, sanitized |
| `locks[].script` | string | ✔ | relative path under `checks/`, must exist |
| `locks[].timeout_sec` | int |  | default 10, max 60 |
| `hints[]` | list | ✔ | 1–6 items, each `{file: hints/NN.md}`, files must exist, each ≤ 4 KiB |
| `solution.walkthrough` | string | ✔ | `solution.md` |
| `solution.script` | string | ✔ | `solution.sh` |
| `resources.memory_mb` | int |  | default 512, max 2048 |
| `resources.cpus` | float |  | default 1.0, max 2.0 |
| `resources.pids` | int |  | default 256, max 1024 |
| `mounts.tmpfs[]` | list |  | ≤2 items `{path: /abs, size_mb: ≤256, nr_inodes: optional}`; total ≤ 512 MB; paths distinct and non-nested (neither may be a prefix of the other), never `/` or under `/dungeon` |
| `network` | enum |  | v1: only `none` (default). Other values reserved, validation error |

Normative validation rules (structural + semantic + security) are itemized in issue 08;
the JSON Schema file lives at `docs/schemas/scenario-v1.schema.json` (authored in issue 08, kept in sync).

### 6.3 Lock scripts (checks)

- POSIX `sh` scripts, run **from the host** via `docker exec` as **root**, piped over stdin
  (never baked into the image, never on the container filesystem — see §7.6).
- Exit 0 = lock OPEN (check passed); non-zero = CLOSED.
- stdout/stderr up to 4 KiB captured; the last line starting with `MSG:` is shown to the player
  as the lock's failure message (sanitized). Everything else goes to the debug log only.
- Must be read-only with respect to game state: a check must not fix or further break the room.
  (Convention, enforced by content review + cookbook; not technically enforceable.)
- Must terminate within `timeout_sec`; on timeout the lock is CLOSED with a timeout message.

### 6.4 `solution.sh` (solvability gate, ADR-006)

- POSIX sh, run as root inside a **fresh** container of the scenario via the same stdin-pipe
  mechanism, in CI and by `scenario test`.
- Contract: after it exits 0, every lock must report OPEN.
- It encodes the *intended* fix; it is never shipped inside the image and never shown by the
  game UI (the human-readable `solution.md` is what `solution` prints).

### 6.5 Hints and solution texts

Markdown, rendered to the terminal through the sanitizer (§10.5) with minimal styling
(bold/emphasis only). Hints must be ordered from nudge → concrete. `solution.md` must contain:
symptom recap, root cause, fix commands, and "lesson learned" section.

### 6.6 Scenario image rules (cookbook, issue 12 — summary)

- Base image: `debian:bookworm-slim` pinned **by digest** (single digest shared by all bundled
  scenarios, documented update procedure). Alpine allowed only with justification.
- Must build and run on `linux/amd64` **and** `linux/arm64`; no arch-conditional binary downloads;
  packages via `apt-get` only, with `--no-install-recommends`, cleaned lists.
- No network use at **runtime** (network is `none`); network at **build** time only via apt/official repos.
- `ENTRYPOINT ["/dungeon-boot.sh"]` pattern: start scenario services (plain processes or loop
  supervisors), then `exec sleep infinity`. No systemd. Docker `Init: true` (tini) reaps zombies.
- Breakage is baked at build time (layers may show the recipe — acceptable; the cookbook
  recommends consolidating breakage into one layer to reduce casual spoilers).
  **Exception:** anything under a spec `mounts.tmpfs` path must be created by `/dungeon-boot.sh`
  at runtime — a tmpfs mounted at container create shadows whatever the image baked at that path.
- Uncompressed image size budget: ≤ 500 MB (floor 5 exceptions up to 700 MB, justified).
- Must not contain: hints, solution files, lock scripts, or the string content of future hints.
- Every scenario must be solvable under the **fixed security profile** (§10.3): no capability
  additions. Cookbook lists breakage classes known to be feasible/infeasible (e.g., iptables and
  `mount -o remount` puzzles are infeasible by design).

---

## 7. Container & Docker Integration

### 7.1 Docker client

- SDK: `github.com/docker/docker/client` with `client.FromEnv` + API version negotiation.
- No shelling out to `docker` CLI (single code path, better errors, no PATH trust issues).
- All created objects labeled: `com.debugdungeon.managed=true`, `com.debugdungeon.scenario=<id>`,
  `com.debugdungeon.run=<run-id>`, `com.debugdungeon.version=<cli-version>`.

### 7.2 Image lifecycle

```
ensureImage(scenario):
  tag = debugdungeon/scn-<id>:<content-hash>     # hash over scenario dir (issue 09)
  if daemon has tag → return
  materialize embedded build context → tmpdir (0700, cleaned via defer)
  docker build (Dockerfile default; platform = daemon native; pull base by digest)
  stream build log → debug log; on failure show last 30 lines + guidance
```

- Content hash = SHA-256 over a deterministic walk (sorted relative paths + mode bit class + file
  bytes) of the scenario directory, excluding `solution.md`, `hints/`, `solution.sh`
  (docs-only changes must not invalidate the image cache). First 12 hex chars used in the tag.
- `clean --all` removes managed images; normal `clean` only removes containers.

### 7.3 Container lifecycle

States (engine-internal, stored in `run.json`):

```
(none) → creating → running
              running → checking → running        (locks failed)
              running → checking → escaped (terminal) → container removed
              running → given_up  (terminal)      → container removed
              running ↔ resetting                 (new container, same run)
              broken  → resetting → running
              running|checking|resetting → broken (container died / daemon lost)
```

"Paused" is presentation language, not a persisted state: when the shell detaches, the run simply
stays `running` with no attached session. Command eligibility by state:

| Command | running | broken | checking/resetting (transient) |
|---|---|---|---|
| `play` (resume) | ✔ attach | ✔ offers reset | retry after transient |
| `check` | ✔ | ✖ (reset first) | busy error |
| `hint` | ✔ | ✔ | busy error |
| `reset` | ✔ | ✔ | busy error |
| `give-up` | ✔ | ✔ | busy error |

Transitions are persisted before/after each Docker call so a crashed CLI can recover
(`play`/`check`/etc. reconcile `run.json` against actual Docker state at startup; issue 20).
**Crash recovery rule:** a run persisted in any *transient* state (`creating`, `checking`,
`resetting`) whose container is actually `running` is recovered to `running` on the next command —
transient states are never durable, so a CLI crash mid-check/mid-reset can never wedge a run as
permanently busy. If the container is gone or not running, the run becomes `broken`.

### 7.4 Container creation parameters (fixed, not scenario-overridable)

| Parameter | Value |
|---|---|
| `Image` | scenario image tag |
| `Entrypoint` | as built (`/dungeon-boot.sh`) |
| `HostConfig.NetworkMode` | `none` |
| `HostConfig.CapDrop` | `ALL` |
| `HostConfig.CapAdd` | `CHOWN, DAC_OVERRIDE, FOWNER, SETGID, SETUID, SETPCAP, KILL, NET_BIND_SERVICE` |
| `HostConfig.SecurityOpt` | `no-new-privileges:true` (default seccomp & AppArmor/SELinux kept) |
| `HostConfig.Privileged` | `false` (never) |
| `HostConfig.Memory` / `NanoCPUs` / `PidsLimit` | from spec, clamped to §6.2 maxima |
| `HostConfig.Init` | `true` (tini zombie reaping) |
| `HostConfig.Binds` / `Mounts` | **no host mounts ever**; only spec tmpfs mounts (§6.2) via `HostConfig.Tmpfs: map[path]"size=<n>m[,nr_inodes=<k>]"` (default tmpfs mode/ownership; scenarios chown/chmod in boot script if needed) |
| `HostConfig.ReadonlyRootfs` | `false` (players must edit the system) |
| `HostConfig.RestartPolicy` | `no` |
| `StopTimeout` | 5s |

### 7.5 Interactive session and sentinel exit codes (ADR-003)

- Session = `docker exec` with TTY: `Cmd = ["/bin/bash", "-l"]` (entry.shell is bash-only in
  schema v1, §6.2), `User = entry.user`,
  `WorkingDir = entry.workdir`, `Env` includes `HISTFILE=<home>/.dungeon_history`,
  `DEBUGDUNGEON=1`, `TERM` passthrough.
- Host puts local stdin into raw mode (`x/term`), streams to the exec attach, handles
  `SIGWINCH` → `ExecResize`. Ctrl-C/Ctrl-Z pass through to the container shell.
- On exec exit, host inspects the exec's exit code:
  - `42` → run locks. All open → victory flow. Any closed → print closed locks, start new exec.
  - `43` → reveal next hint (or "no more hints"), start new exec.
  - `44` → confirm on host side (`y/N`), then give-up flow; if declined, re-enter.
  - any other → pause run, print resume instructions.
- Helper scripts `escape`, `hint`, `giveup` are ~3-line sh scripts that `exit 42|43|44`.
  Because a plain `exit 42` in the player shell triggers the same path, the protocol treats the
  *exit code as the request* — no ambiguity, no IPC channel, no extra attack surface.

### 7.6 Helper injection

At container create time (before start), the engine copies a tar archive to `/` via
`CopyToContainer` containing:

- `/dungeon/bin/escape`, `/dungeon/bin/hint`, `/dungeon/bin/giveup` (mode 0755)
- `/dungeon/.motd` (rendered, sanitized lore banner; printed by shell profile snippet)
- `/etc/profile.d/zz-debugdungeon.sh` → prepends `/dungeon/bin` to PATH, prints motd once per session

Scenarios that sabotage PATH remain playable: the banner always names the absolute paths
(`/dungeon/bin/escape`). The cookbook forbids scenarios from deleting `/dungeon`.

### 7.7 Lock execution

For each lock, host runs a non-TTY exec: `Cmd = ["timeout", "<timeout_sec>", "/bin/sh", "-s"]`
(coreutils `timeout` runs **inside** the container and reliably kills the script's process group —
the Docker API cannot kill an exec), `User = "root"`, `AttachStdin`, writes the script bytes,
closes stdin, reads multiplexed stdout/stderr (≤ 4 KiB kept). Host waits with a
`timeout_sec + 2s` backstop context; if even the backstop trips (in-container `timeout` gone or
wedged), report CLOSED with a timeout message, log a stray-process warning, and rely on
`reset`/teardown for cleanup. Cookbook forbids scenarios removing coreutils (`timeout` included).
Locks run sequentially in file order (v1), results collected into
`LockReport{id, name, open, msg, durationMs}`.

### 7.8 Cleanup & GC

- Terminal states remove the container (`force=true`).
- `clean`: remove containers labeled `com.debugdungeon.managed=true` not referenced by `run.json`
  (+ with `--all`: also managed images, sparing the active run's image). `clean` never touches
  `run.json` itself — only `give-up`/victory terminate runs.
- `doctor` warns when managed leftovers or > N GB of managed images exist.

---

## 8. Local State & Storage

### 8.1 Path layout

| Purpose | macOS | Linux |
|---|---|---|
| State root | `~/Library/Application Support/debugdungeon/` | `${XDG_DATA_HOME:-~/.local/share}/debugdungeon/` |
| Config file | `<state root>/config.yaml` | `${XDG_CONFIG_HOME:-~/.config}/debugdungeon/config.yaml` |
| Progress | `<state root>/progress.json` | `<data>/progress.json` |
| Active run | `<state root>/run.json` | `<data>/run.json` |
| Installed packs | `<state root>/packs/<pack-name>/` | `<data>/packs/<pack-name>/` |
| Debug logs | `~/Library/Logs/debugdungeon/debug.log` | `${XDG_STATE_HOME:-~/.local/state}/debugdungeon/debug.log` |

`DEBUGDUNGEON_HOME` (or `--data-dir`) overrides everything: the per-OS table above is replaced by
a single root with the fixed layout `config.yaml`, `progress.json`, `run.json`, `run.lock`,
`history.jsonl`, `packs/`, `logs/debug.log` — used by tests and E2E. Directories 0700, files 0600.
Log rotation: truncate at 5 MB keeping one `.old`.

### 8.2 `progress.json` (schema_version 1)

```json
{
  "schema_version": 1,
  "scenarios": {
    "rusty-path": {
      "status": "cleared",            // new | cleared | given_up
      "clears": 1,
      "best_time_sec": 462,
      "hints_used_best": 1,
      "resets_best": 0,
      "first_cleared_at": "2026-07-10T12:00:00Z",
      "last_played_at": "2026-07-10T12:00:00Z"
    }
  },
  "totals": { "clears": 1, "playtime_sec": 462 }
}
```

### 8.3 `run.json` (schema_version 1)

```json
{
  "schema_version": 1,
  "run_id": "r-8f3a2c",
  "scenario_id": "rusty-path",
  "source": "bundled",                 // bundled | pack:<name>
  "content_hash": "ab12cd34ef56",
  "image_ref": "debugdungeon/scn-rusty-path:ab12cd34ef56",
  "container_id": "…",
  "state": "running",
  "started_at": "2026-07-10T11:52:18Z",
  "hints_revealed": 1,
  "resets": 0,
  "last_lock_report": [ {"id":"path-restored","open":false,"msg":"…"} ]
}
```

### 8.4 Durability rules

- All writes: marshal → temp file in same dir → `fsync` → atomic `rename`.
- On unreadable/corrupt JSON: move aside to `<name>.corrupt-<ts>`, start fresh, warn once.
- Unknown `schema_version` > supported: refuse with "upgrade debugdungeon" error (never destroy).
- Time source: wall clock UTC (RFC3339); elapsed = now − started_at (documented simplification).

---

## 9. Game Rules (normative)

1. A run starts at `play` when no run exists for that scenario (image ensured, container created).
2. The run clock never pauses and never resets (including `reset`) until terminal state.
3. `reset` destroys the container, creates a fresh one from the same image, keeps
   `hints_revealed`, increments `resets`.
4. Victory requires **all** locks OPEN in a single `check`/`escape` evaluation.
5. On victory: progress updated (`status=cleared`, best-time min-merge, counters), run + container removed.
6. `give-up`: progress `status=given_up` (unless already `cleared`), solution revealed unless `--no-reveal`, run + container removed.
7. Replays allowed always for unlocked scenarios; `cleared` never downgrades (a later give-up on a replay keeps `cleared`).
8. Floor gating per §3.5; `solution <id>` requires that scenario `cleared` or `given_up`.
9. Hints reveal strictly in order; revealing is idempotent per index; total persisted on the run and merged to progress at terminal state.

---

## 10. Security Model

### 10.1 Assets and attacker models

Assets: the player's host machine (files, credentials, network position), the player's Docker
daemon, integrity of released binaries, the player's trust in installed content.

| Attacker | Vector | Addressed in |
|---|---|---|
| A1 Malicious scenario author (community pack) | crafted `scenario.yaml`, Dockerfile, check/solution scripts, hint text | §10.3–10.6, issues 08, 11, 59–61 |
| A2 Compromised dependency / base image | supply chain | §10.7, issues 02, 36, 38 |
| A3 Local opportunist (other local process/user) | state files, socket misuse | §8.4 perms; Docker socket is out of scope (OS-level trust) |
| A4 Player themselves (cheating) | reading images/checks | Non-goal (§2.3) |

### 10.2 Trust boundaries (numbered; referenced by issues)

| # | Boundary | Rule |
|---|---|---|
| TB1 | CLI ↔ Docker daemon | Daemon is trusted infra. CLI must function without root when the user's daemon setup allows it; never instruct `sudo` silently |
| TB2 | Host ↔ scenario container | Container is a sandbox with the fixed profile (§7.4). Nothing from inside executes on the host. The only signals crossing inward→outward are exec exit codes and captured byte streams (always treated as untrusted data). **Residual risk:** during the interactive raw-PTY session, container programs write directly to the player's terminal unsanitized (that is the product); hostile content could emit terminal control sequences there. Accepted for bundled (reviewed) content; for community packs this risk is named in the trust gate (§10.6) |
| TB3 | Engine ↔ scenario content | All scenario-sourced strings are untrusted input: strict schema, size caps, sanitization before terminal output |
| TB4 | Bundled vs community content | Bundled = reviewed in-repo (trusted at build). Community packs = untrusted until user consents through the trust gate (§10.6) |
| TB5 | Release pipeline ↔ users | Reproducible-ish builds, checksums, provenance (§10.7) |

### 10.3 Container secure defaults (fixed profile)

Exactly §7.4. Additional invariants, enforced by code + regression tests (issue 35):

- No host bind mounts, no volumes, no device mappings, no host network/PID/IPC namespaces, ever.
- Capabilities: fixed allowlist; scenarios cannot request more (schema has no field for it — capability creep requires a schema change + ADR).
- Player CLI never runs `docker` with elevated privileges and never modifies daemon config.

Residual risk (documented for users): running any container executes scenario-controlled code
inside the sandbox on your machine; kernel 0-days in container isolation are out of scope, which
is one reason community packs get an explicit consent gate.

### 10.4 Input validation summary (parser boundaries)

| Input | Validation (issue) |
|---|---|
| `scenario.yaml` | strict YAML (no unknown fields), field constraints table §6.2, path containment, symlink rejection (07, 08) |
| Scenario file tree | all referenced paths must resolve within the scenario root; reject absolute/`..`/symlinks pointing outside; per-file and total size caps (08) |
| Pack archives (tar/git) | safe extraction: reject `..`, absolute paths, hardlinks/symlinks escaping root, size & file-count quota, no setuid bits preserved (59) |
| progress/run JSON | schema_version check, defensive decode, corrupt-quarantine (§8.4) (20, 26) |
| Lock/exec output | length-capped, sanitized before display (11, 19) |
| CLI args | cobra typed flags; scenario id regex before any FS/docker use (04, 21) |

### 10.5 Terminal output sanitization (TB3)

All strings originating from scenario content or container output are passed through
`textsafe.Sanitize` before writing to the user's TTY **outside raw shell mode**:
strip all C0/C1 control chars except `\n`/`\t`, neutralize ANSI CSI/OSC sequences
(including OSC 8 hyperlinks, title-set, clipboard OSC 52), enforce UTF-8 validity, cap length.
Rationale: escape-sequence injection from hints/lore/check messages is the most realistic attack
from content (terminal spoofing/clipboard write). The interactive shell itself is exempt by
design (the player asked for a real TTY into the sandbox) — documented residual risk.

### 10.6 Community pack trust gate (Wave 8)

- Install requires explicit source; nothing auto-fetches.
- On install: full validation (as bundled) + a **capability disclosure**: image base, tmpfs sizes,
  resource asks, per-file listing summary; user must type the pack name to confirm.
- **Consent covers build execution.** Building a pack scenario runs its Dockerfile in the Docker
  daemon's build environment, which (unlike the runtime profile) has network access for base-image
  pulls and package installs. The disclosure states this in plain words, and the raw-PTY residual
  risk (TB2). **Ordering guarantee:** no `docker build` of pack content ever happens before BOTH
  install consent and the pack's first-play acceptance are recorded.
- Provenance recorded (`source`, commit hash for git, sha256 for tarballs) in
  `packs/<name>/.provenance.json`; `list` shows origin; first `play` of any pack scenario
  re-shows a one-line origin warning.
- Packs never extend engine capabilities: same schema, same fixed container profile.

### 10.7 Supply chain & release security

- `go.sum` enforced; Dependabot (gomod + actions); `govulncheck` in CI.
- GitHub Actions: pinned by commit SHA, least-privilege `permissions:` blocks, no `pull_request_target` with checkout of PR code.
- Base image pinned by digest; documented rotation procedure.
- Releases: goreleaser → archives + `checksums.txt`; artifact attestation (GitHub `attest-build-provenance`) — signing (cosign) tracked as v2 (KU-6).
- SECURITY.md: private vulnerability reporting via GitHub advisories, 90-day disclosure target.

### 10.8 Privacy

**No implicit network I/O** from the CLI (ADR-007). Every network operation in the system is
user-initiated and enumerable:

1. Docker daemon pulls pinned base images during `docker build` (triggered by `play`/`scenario test`).
2. `pack install <git-url>` runs the system `git` at the user's explicit command (Wave 8).
3. The opt-in update check (Wave 9): off by default, config-gated, GET-only, ≤1/24h, no
   identifiers beyond the User-Agent version string.

Nothing else touches the network — no telemetry, no crash upload, no implicit update checks.
No file outside the state root and Docker is written.

---

## 11. Failure Modes & Error UX

| # | Condition | Detection | Player-facing behavior |
|---|---|---|---|
| F1 | Docker daemon unreachable | ping on first docker use | exit 3; per-OS guidance (Docker Desktop/OrbStack/colima start hints); `doctor` reference |
| F2 | Daemon API too old | version negotiation < 1.44 | exit 3 with found/required versions |
| F3 | Base image pull fails (offline/registry down) | build error class | "network needed for first build of this room; retry when online" |
| F4 | Image build fails | non-zero build | last 30 log lines + "report bug" template link (bundled content should never fail = CI-guaranteed) |
| F5 | Container dies mid-run (OOM, crashed boot) | exec attach error / inspect on `play`/`check` | state → `broken`; offer `reset` (keeps hints/clock) |
| F6 | Lock script timeout | exec wait > timeout | lock CLOSED with "timed out after Ns" |
| F7 | `run.json` references missing container | reconcile at startup | state → `broken`, same as F5 |
| F8 | Corrupt state JSON | decode error | quarantine + fresh file + warning (§8.4) |
| F9 | Two CLIs race on one run | short-scoped flock around each load-mutate-save critical section (`run.lock`); the interactive session does NOT hold the lock while the shell is attached | concurrent mutators serialize; a second mutator blocked > 2s errs: "another debugdungeon command is mid-operation — retry in a moment". Second-terminal `check`/`hint`/`status` remain possible during play (§3.3) |
| F10 | Disk pressure from images | `doctor` reports managed image totals (Docker `DiskUsage` API) | warn + suggest `clean --all` |
| F11 | Terminal without TTY (`play` in pipe) | `IsTerminal` check | exit 2: `play` requires an interactive terminal; `check` works headless |
| F12 | Unsupported daemon arch (e.g. Windows containers) | `Info.OSType != "linux"` | exit 3 with explanation |

Error rendering: one-line summary + optional detail + "next command to try". No stack traces
except with `--verbose` (issue 06).

---

## 12. Testing & QA Strategy

| Layer | What | Where it runs |
|---|---|---|
| Unit | spec parsing/validation table-driven; content hash vectors; sanitizer corpus (ANSI/OSC/UTF-8 attacks); progress merge logic; unlock rules | every PR (no Docker) |
| Validator negative corpus | `testdata/badscenarios/*` each with expected error code (path escape, symlink, oversize, unknown field, bad enum…) | every PR |
| Docker integration | dockerx build/create/exec against real daemon; security profile assertion test: inspect created container's HostConfig/caps and compare to §7.4 golden | Linux CI job with Docker (and locally via `make itest`) |
| Solvability matrix (ADR-006) | for every bundled scenario: build → run `solution.sh` → all locks OPEN; also assert locks CLOSED on pristine container (breakage really present, catches "already-open" locks) | CI matrix `linux/amd64` + `linux/arm64` (ubuntu-24.04-arm runner), path-filtered to changed scenarios + full weekly run |
| E2E session | PTY-driven playthrough of `welcome-cell` via the real binary (expect-style: banner → `hint` → fix → `escape` → victory; then `list` shows cleared) | Linux CI job |
| Security regression | negative container-profile tests (privileged/net/host-mount impossible), pack extraction attack corpus (Wave 8), `govulncheck` | every PR |
| Release smoke | goreleaser `--snapshot` build all platforms; `debugdungeon version/doctor` runs in a clean container | release PRs + tags |

CI note: scenario jobs must prune Docker state between matrix items to respect runner disk (~14 GB).

---

## 13. Distribution & Release

- goreleaser: darwin/{arm64,amd64}, linux/{arm64,amd64}; CGO off; `-trimpath`; version/commit/date via ldflags.
- Channels: GitHub Releases (tar.gz + checksums + attestation), Homebrew tap `Saber5656/homebrew-tap` (formula auto-bumped by goreleaser), `go install` works as fallback.
- Versioning: SemVer, `v0.x` during MVP; `v1.0.0` = MVP release gate (§2.1). Conventional commits recommended, not enforced.
- Scenario content ships inside the binary (go:embed, ADR-004) — one artifact, no content path issues. Binary size budget ≤ 30 MB (content is text + small files; images build on demand).
- WSL2: documented as supported-best-effort (needs Docker reachable from WSL). No windows/amd64 binary in v1 (KU-3).

---

## 14. Content Plan (bundled scenarios)

Common: every scenario teaches a named skill, has 2–4 hints, `solution.md` with "lesson learned",
`solution.sh`, and 1–3 locks. IDs are stable API. ★ = difficulty.

### Floor 1 — The Entrance Halls (★1, Wave 4 = MVP)

| ID | Room | Breakage (baked) | Locks (all root-run) |
|---|---|---|---|
| `welcome-cell` | tutorial | a `LOCKED` marker file + a note teaching `hint`/`escape`; player deletes marker and lights a torch per instructions | `stone-removed` (marker absent); `torch-lit` (player-created file) |
| `rusty-path` | The Rusty PATH | `/etc/profile` sets PATH missing `/usr/local/bin`; gate binary lives there; decoy broken `open-gate` in `/opt/decoy` earlier in PATH | login-shell PATH contains `/usr/local/bin` before decoy; `/var/dungeon/gate-opened` exists (created by running `open-gate`) |
| `forbidden-scroll` | permissions | app user `scribe` can't read `/etc/scroll/config.yaml` (root:root 0600) + parent dir 0700; service script fails | `su -s /bin/sh scribe -c 'cat …'` succeeds; scroll service writes heartbeat file |
| `broken-symlink` | dangling links | `/etc/app/current` symlink → deleted release dir; two versioned dirs exist (one corrupt marker) | symlink resolves to the good release; app script outputs OK |

### Floor 2 — The Daemon Warrens (★2, Wave 6)

| ID | Room | Breakage | Locks |
|---|---|---|---|
| `sleeping-daemon` | crashloop | boot supervisor loops `heartd` which exits on invalid `/etc/heartd.conf` (bad key + wrong pidfile dir) | `heartd` process alive > 10s; heartbeat file fresh |
| `port-poltergeist` | port conflict | rogue process binds :8080 before the real `wardd`; boot order race scripted | `wardd` listening on 8080; rogue absent |
| `zombie-horde` | runaway respawner | cron `* * * * *` spawns zombie workers that multiply (bounded by pids limit); a similarly-named legit worker must survive | zombie count = 0; cron source removed; legit `grave-worker` still alive |
| `cron-curse` | cron env | job needs `RITUAL_HOME` env + absolute path, and has an unescaped `%` in its crontab line; silently produces nothing | fresh artifact file exists (retry-checked); crontab line sound (no bare `%`, absolute path) |

### Floor 3 — The Flooded Archives (★3, Wave 6)

| ID | Room | Breakage | Locks |
|---|---|---|---|
| `bloated-vault` | disk full | 64 MB tmpfs at `/var/vault` filled by runaway `debug.log`; app can't write | free space ≥ 20%; app write test passes; log growth stopped (offender loop disabled) |
| `inode-imp` | inode exhaustion | tmpfs `nr_inodes=8192` exhausted by session-file spam | free inodes ≥ 50%; session probe works; spammer cron disabled |
| `log-labyrinth` | needle in logs | service fails at boot with misleading generic error; true cause (bad locale in conf) buried across rotated logs | service healthy; player wrote root-cause filename into `/root/answer` (grader greps expected token) |

### Floor 4 — The Tangled Battlements (★3–4, Wave 6; all localhost, network:none)

| ID | Room | Breakage | Locks |
|---|---|---|---|
| `dns-demon` | name resolution | app connects to `vault.internal` → `/etc/hosts` poisoned to wrong loopback alias + `nsswitch.conf` order broken | `getent hosts vault.internal` → 127.0.0.1; app health endpoint 200 via curl |
| `nginx-gargoyle` | reverse proxy | nginx conf: syntax error + upstream port mismatch to local backend | `nginx -t` passes; nginx running; `curl localhost:80/` returns backend marker |
| `certificate-crypt` | expired TLS | local https service uses expired self-signed cert (baked with past `-days`); client refuses | new cert valid ≥ 30 days & CN match; `curl --cacert …` succeeds |
| `proxy-maze` | env poisoning | `http_proxy`/`NO_PROXY` garbage in `/etc/environment` + profile breaks local HTTP client | client script fetches localhost OK in fresh login shell; poison vars absent |

### Floor 5 — The Data Crypts (★4, Wave 6)

| ID | Room | Breakage | Locks |
|---|---|---|---|
| `postgres-lich` | pg won't start | `postgresql.conf` bad param + `pg_hba.conf` rejects app user | pg accepting connections; app user can SELECT over localhost |
| `redis-wraith` | AOF corruption | truncated appendonly file; redis refuses to start | redis PING ok; sentinel key present (from pre-corruption data); AOF loads clean |
| `migration-mimic` | half-applied migration | app schema at v3-partial (missing column + stray lock row in migrations table) | migration table consistent; app smoke query works |
| `secrets-specter` | secret drift | DB password already rotated server-side; app still reads the old value from one of three config sources; teaches hygienic rotation | app connects; old secret purged everywhere; new secret file 0640 root:appgroup and never logged |

### Capstone (★5, Wave 6)

| ID | Room | Breakage | Locks |
|---|---|---|---|
| `final-cascade` | The Cascade Throne | chained: disk-full → service down → stale pid + perms → app 500 (3 faults across layers, fixing order matters) | 4 locks: disk, service, app 200, root-cause note |

---

## 15. Roadmap (waves → issues)

| Wave | Theme | Issues | Release |
|---|---|---|---|
| 0 | Repo & CLI foundation | 01–06 | — |
| 1 | Scenario model & safety | 07–12 | — |
| 2 | Container engine | 13–19 | — |
| 3 | Game loop | 20–28 | — |
| 4 | Solvability gate + MVP content | 29–33 | — |
| 5 | MVP hardening & release | 34–39 | **v1.0.0 (MVP)** |
| 6 | Content expansion (16 rooms) | 40–55 | v1.x |
| 7 | Authoring kit | 56–58 | v1.x |
| 8 | Community packs | 59–61 | v1.x |
| 9 | Gamification & polish | 62–66 | v1.x |

`docs/ISSUE_PLAN.md` is the normative ordering/dependency source.

---

## 16. Known Unknowns

| # | Unknown | Impact | Plan |
|---|---|---|---|
| KU-1 | GH Actions arm64 runner availability/limits for the solvability matrix | CI cost/coverage | verify in issue 29 (first workflow that needs it); fallback = amd64-only PR gate + weekly arm64 |
| KU-2 | Podman socket compatibility gaps (exec resize, CopyToContainer) | user support cost | best-effort; doctor detects and warns; test once in Wave 5 |
| KU-3 | WSL2 UX (PTY + Docker Desktop integration) | Windows reach | manual test during Wave 5; document |
| KU-4 | `debian:bookworm-slim` digest freshness vs CVE churn | image hygiene | rotation procedure in cookbook; Dependabot doesn't cover digests → quarterly manual chore issue |
| KU-5 | PostgreSQL/Redis apt install size vs 500 MB budget on arm64 | Floor 5 feasibility | measured in issue 51/52; budget exception pre-approved to 700 MB |
| KU-6 | cosign signing UX for casual users | release trust | v1 ships checksums + GitHub attestation; cosign revisit v2 |
| KU-7 | Exit-code collision: player program legitimately exits 42 | spurious lock run | acceptable (lock run is harmless); documented easter egg |
| KU-8 | Homebrew tap name/formula availability | distribution | verify at issue 36 |
| KU-11 | Service patterns (su login, cron, nginx, postgres, redis) unproven under the fixed profile before content fan-out | Floor 2–5 feasibility | issue 29 ships feasibility probe fixtures per pattern, run in the harness suite before Wave 6 starts |

---

## 17. Glossary

| Term | Meaning |
|---|---|
| Room / Scenario | One broken environment definition (directory) |
| Floor | Themed difficulty group of rooms |
| Lock | One automated check that must pass to escape |
| Run | One attempt-session at a scenario (container + counters) |
| Escape | All locks open in one evaluation |
| Pack | Installable directory of community scenarios |
| Bundled | Scenarios embedded in the released binary |
