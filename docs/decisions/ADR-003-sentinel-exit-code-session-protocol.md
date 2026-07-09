# ADR-003: Single-terminal session protocol via sentinel exec exit codes

- Status: Accepted
- Deciders: Fable (design), reviewable by owner

## Context

The player lives inside an interactive `docker exec` shell. Game actions (`escape`, `hint`,
`giveup`) must reach the host CLI while the shell has the terminal. Candidate channels:
a second terminal only; a unix socket/FIFO bridged into the container; file markers polled by
the host; or terminating the exec with distinguished exit codes.

## Decision

In-room helper commands are 3-line scripts that **exit the shell with sentinel codes**:
`42` escape-attempt, `43` next-hint, `44` give-up. The supervising host process inspects the
exec exit code, performs the action, and (unless the run ended) **starts a fresh exec session**
back into the same container. Any other exit code pauses the run. `check`/`hint` remain available
from a second terminal through identical code paths.

## Consequences

- Zero extra IPC surface across the trust boundary (TB2): the only inbound signal is an integer
  exit code, which is treated as a request, never as data.
- Trivially implementable and testable; no polling, no socket lifecycle.
- Cost: re-entry resets the working directory and shell process state (history preserved via
  `HISTFILE`); documented quirk (DESIGN §3.3).
- Collision: a player process exiting 42 organically triggers a harmless lock evaluation (KU-7).

## Alternatives considered

- **FIFO/socket bridge**: richer (hints without leaving shell) but adds an attack surface and
  lifecycle complexity inside an environment the scenario intentionally breaks. Rejected for v1.
- **Marker-file polling**: racy, needs a poller, and scenario breakage (disk full!) can wedge it.
- **Second terminal only**: robust but clunky; kept as a *supplement*, not the primary UX.
