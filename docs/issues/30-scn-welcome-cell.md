# Title

Scenario: welcome-cell (Floor 1 tutorial)

## Summary

Build the tutorial room that teaches the game verbs (`hint`, `escape`, reading the room) with a
trivial two-lock puzzle. Also fixes the shared base-image digest for all bundled content.

## Context

First contact with the product for every player; must be unlosable and fast (< 10 min). This issue
sets `BASE` digest in the cookbook (12 left it as a placeholder).

## Scope

- `scenarios/welcome-cell/` complete per DESIGN §6.1
- Update `docs/COOKBOOK.md` §2 with the chosen multi-arch `debian:bookworm-slim` digest
- Not: engine changes

## Detailed Requirements

1. `scenario.yaml`: id `welcome-cell`, floor 1, difficulty 1, topics `[basics, filesystem]`,
   time_estimate_min 10, defaults for entry (root/bash//root — workdir `/root`), network none,
   default resources. Lore (≤ 600 chars), e.g.: awakening in a cell; a note on the floor; the
   stone that seals the door is cursed; light the torch and remove the stone to walk free.
   Locks:
   - `stone-removed` / name `The sealing stone is gone` / `checks/stone.sh` / timeout 10
   - `torch-lit` / name `The torch is lit` / `checks/torch.sh` / timeout 10
2. `image/Dockerfile` (pattern per cookbook; breakage in one layer):
   - `ARG BASE=debian:bookworm-slim@sha256:<digest>` — implementer picks the current multi-arch
     manifest digest, verifies it resolves on amd64 AND arm64 (`docker buildx imagetools inspect`),
     records it in COOKBOOK §2.
   - Install nothing (slim already has coreutils + bash? — bookworm-slim includes bash: verify;
     if absent, `apt-get install -y --no-install-recommends bash` then clean lists).
   - Create `/var/dungeon/` ; write `/var/dungeon/LOCKED` (the stone; content: `an ancient seal`).
   - Write `/root/READ-ME-FIRST.txt`:
     instructions teaching: look around (`ls`), read (`cat`), the three game commands, and the two
     tasks — `rm /var/dungeon/LOCKED` and `echo lit > /var/dungeon/torch`. Explicit spoilers are
     CORRECT here (tutorial).
   - `image/dungeon-boot.sh`: writes `/var/dungeon/boot-ok`, `exec sleep infinity`. ENTRYPOINT it.
3. `checks/stone.sh`:
   ```sh
   if [ -e /var/dungeon/LOCKED ]; then echo "MSG: The sealing stone still blocks the door."; exit 1; fi
   exit 0
   ```
4. `checks/torch.sh`: `/var/dungeon/torch` must exist and contain `lit`
   (grep -qx allowed; MSG: `The cell is pitch dark. The note mentioned a torch…`).
5. `hints/01.md`: look around with `ls`, read the note with `cat /root/READ-ME-FIRST.txt`.
   `hints/02.md`: the exact two commands. (2 hints total — tutorial.)
6. `solution.md`: symptom recap, the two commands, Lesson Learned (filesystem verbs; the game
   loop: read the room → change it → `escape`).
7. `solution.sh`: the two commands, idempotent (`rm -f`, `echo lit > …`).
8. Cookbook checklist (12 §14) executed and pasted in PR.

## Acceptance Criteria

- [ ] `ValidateDir` zero violations; registry loads it; `list` shows it on Floor 1.
- [ ] Solvability harness (29) passes on amd64 and arm64: pristine has ≥1 closed lock; solution opens all.
- [ ] Image ≤ 150 MB uncompressed; builds < 60s warm-cacheless on CI.
- [ ] Manual playthrough transcript: fresh player path incl. one `hint`, failed `escape` (only stone removed) showing the torch MSG, then success.
- [ ] COOKBOOK §2 digest recorded and used via `ARG BASE`.

## Validation

`make itest-scenarios DD_SCENARIO_FILTER=welcome-cell` green (both arches via CI); transcript attached.

## Dependencies

12, 29 (playable end-to-end needs 21/22 — manual QA notes may use the harness alone if the loop isn't merged yet).

## Non-goals

Multi-language notes, ambience art (65), difficulty balance beyond "unlosable".

## Design References

DESIGN §14 Floor 1, §3.2 (transcript example), §6; COOKBOOK (12).
