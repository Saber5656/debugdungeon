# Title

Container lifecycle with the fixed hardened security profile

## Summary

Implement room-container create/start/stop/remove in `internal/dockerx` with the **fixed,
non-overridable security profile** of DESIGN §7.4, expressed in one auditable function with a
golden test.

## Context

This is the single most security-critical code path (TB2). The profile is product law: scenarios
cannot request deviations (schema has no field), and issue 35 regression-tests it live.

## Scope

- `internal/dockerx/room.go` + golden/unit/itest
- Not: session exec (17), helper injection (18), GC (16)

## Detailed Requirements

1. `BuildRoomConfig(spec *scenario.Spec, runID, imageRef, cliVersion string) (container.Config, container.HostConfig, warnings []string)`
   — pure function, no I/O, no logging (clamping reports via the returned `warnings`; callers log).
   Values exactly:
   - Config: `Image=imageRef`, `Labels` with full keys `com.debugdungeon.managed="true"`,
     `com.debugdungeon.scenario=spec.ID`, `com.debugdungeon.run=runID`,
     `com.debugdungeon.version=cliVersion`; `Env=["DEBUGDUNGEON=1"]`;
     `StopTimeout: ptr(5)`. Entrypoint/Cmd inherited from image.
   - HostConfig:
     - `NetworkMode: "none"`
     - `CapDrop: ["ALL"]`
     - `CapAdd: ["CHOWN","DAC_OVERRIDE","FOWNER","SETGID","SETUID","SETPCAP","KILL","NET_BIND_SERVICE"]`
     - `SecurityOpt: ["no-new-privileges:true"]`
     - `Privileged: false`; no `Binds`, no `Mounts` (except tmpfs below), no `Devices`,
       `PidMode/IpcMode/UTSMode` empty (private), `PublishAllPorts: false`, `PortBindings: nil`
     - `Resources: { Memory: spec MB→bytes, NanoCPUs: cpus*1e9, PidsLimit: &pids }` — clamp to §6.2
       maxima defensively even though validator enforces (defense in depth; clamping logs a warning)
     - `Tmpfs: map[path]"size=<n>m[,nr_inodes=<k>]"` from spec mounts
     - `Init: ptr(true)`, `RestartPolicy: {Name:"no"}`, `ReadonlyRootfs: false`, `AutoRemove: false`
2. `CreateRoom(ctx, api, spec, runID, imageRef) (containerID string, err error)` →
   `ContainerCreate` with the tuple from (1) (warnings → caller's logger), name
   `dd-<spec.ID>-<runID>`; on a name-conflict error (`errdefs.IsConflict`) retry with suffixes
   `-2` … `-5`, then fail with the conflict error.
3. `StartRoom`, `StopRoom` (timeout 5s), `RemoveRoom(force=true)`, `InspectRoom` (returns running
   state + exit info) — thin wrappers with typed errors.
4. **Golden test**: marshal the (Config, HostConfig) pair for a maximal fixture spec to JSON and
   compare byte-exact to `internal/dockerx/testdata/room_config.golden.json` (pure JSON — no
   comments; the DESIGN §7.4 cross-reference lives in an adjacent
   `room_config.golden.README.md`). Any profile change = golden diff = deliberate review.
5. Unit test: clamping (spec forged with 9999 MB memory post-validator) clamps to 2048 and returns
   a warning naming field + original value.
6. itest: create+start `_template` room; `docker inspect` via API and assert live: CapAdd set
   exactly, NetworkMode none, NNP present, PidsLimit, Init, no mounts other than declared tmpfs;
   then stop+remove. (Issue 35 re-runs this as a permanent security gate; here it proves the code.)

## Acceptance Criteria

- [ ] Golden test in place; changing any HostConfig field fails it.
- [ ] Live inspect assertions pass on a real daemon (itest).
- [ ] No public API of this package accepts capability/network/mount parameters (grep-able:
      "the profile is not a parameter" comment on BuildRoomConfig).
- [ ] Stop respects 5s then kills; Remove force works on running containers.

## Validation

`make test` + `make itest`; attach the golden JSON in the PR for reviewer sign-off against DESIGN §7.4 table.

## Dependencies

07 (Spec types), 13.

## Non-goals

Session/exec (17), tmpfs feasibility for specific scenarios (44/45), rootless-specific quirks (KU-2).

## Design References

DESIGN §7.3, §7.4, §10.3; ADR-002.
