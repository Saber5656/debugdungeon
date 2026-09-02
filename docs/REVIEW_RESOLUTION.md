# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #1. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTN39Ys6QAfnN — sentinel failures reach the watched shell

Finding: A helper child shell that exits 42, 43, or 44 does not necessarily terminate the parent shell being watched by the session protocol.

Normative resolution:
- The sentinel exit codes must be emitted by the watched shell/process whose status the session manager observes, or by a sourced/function-based command that preserves that status.
- A child process may not merely exit with a sentinel while the watched parent continues and reports success.
- The protocol must define precedence when a command both emits output and returns a sentinel, and must retain the exact terminal status.

Focused verification before resolving this thread:
- Run each sentinel path through the same watched-shell mechanism used by the CLI and assert the parent-observed exit code is exactly 42, 43, or 44.
- Include a child-shell counterexample and confirm it cannot be mistaken for a successful watched session.
- Verify normal completion and ordinary command failure remain distinct from the sentinel taxonomy.

## PRRT_kwDOTN39Ys6QAfnR — lock paths are scenario-local

Finding: Lock scripts must use the scenario directory rather than the bundle root, otherwise fixtures can collide or mutate the wrong scenario.

Normative resolution:
- Every lock/read/write operation must derive its path by joining the scenario's declared directory (l.Dir) with the scenario-relative lock path.
- Bundle-root fallback is forbidden unless the scenario contract explicitly names the root and passes containment validation.
- The path must remain within the selected scenario directory after normalization.

Focused verification before resolving this thread:
- Execute two scenarios with same-named locks and assert each operation touches only its own l.Dir.
- Test traversal and absolute-path inputs and require rejection before filesystem mutation.
- Inspect the generated command/fixture paths to confirm no bundle-root lock is created accidentally.

## PRRT_kwDOTN39Ys6QAfnS — crash transient state is not blindly started

Finding: Creating or resetting transient crash state and then starting the scenario without checking injection can produce a false-feasible scenario.

Normative resolution:
- Crash setup must transition through an explicit broken/setup state, recreate or repair the transient state as needed, inject the intended failure, and verify the failure is observable before the scenario is marked startable.
- If setup, recreation, or verification fails, the scenario must be marked broken with an actionable reason; it must not proceed as if injection succeeded.
- State transitions and cleanup must be idempotent and must not hide a stale process or stale crash artifact.

Focused verification before resolving this thread:
- Simulate absent, stale, and already-running transient state and assert the state machine takes the documented repair/recreate path.
- Confirm a failed injection blocks scenario start and records the failure cause.
- Repeat setup after interruption and verify there is no false success or leftover process that bypasses the gate.

## PRRT_kwDOTN39Ys6QAfnV — SSH known_hosts stays inside controlled state

Finding: SSH with accept-new can write to the user's ~/.ssh/known_hosts, which violates the isolated scenario state boundary.

Normative resolution:
- Host-key material must be placed under the scenario's controlled state/temp directory and passed explicitly to the SSH invocation.
- The implementation must not mutate the user's home SSH directory or depend on ambient global known_hosts.
- The state path must have restrictive permissions and deterministic cleanup appropriate to the session lifetime.

Focused verification before resolving this thread:
- Run an SSH connection with an isolated HOME and explicit state path, then assert only the controlled state path changes.
- Test a new host key, a changed host key, and a missing state directory with explicit fail/repair outcomes.
- Scan the user's home path and environment-derived SSH defaults in the test harness to ensure they are not selected.

## PRRT_kwDOTN39Ys6QAfnX — timeout escalation leaves no descendants

Finding: A timeout that sends only TERM can leave child processes alive, especially when the command creates a process tree.

Normative resolution:
- Timeout handling must target the process group or equivalent descendant set, send TERM first, wait for the bounded grace period, then send KILL to the remaining group when necessary.
- The command must await reaping and report timeout distinctly from ordinary command failure.
- Cleanup failure or surviving descendants is a hard failure; the session may not be reported complete.

Focused verification before resolving this thread:
- Start a fixture that forks a child and assert timeout sends the documented escalation sequence.
- After the timeout path, enumerate the process group and require zero surviving descendants.
- Test graceful TERM handling, unresponsive children, and repeated timeout cleanup for deterministic terminal results.

## PRRT_kwDOTN39Ys6QAfnd — feasibility uses a closed lock fixture

Finding: A solvability fixture needs an initially closed lock such as /var/dungeon/SEAL, or an explicit exemption from the initial-breakage gate; otherwise feasibility checks contradict the scenario setup.

Normative resolution:
- The fixture must create the closed lock at the exact scenario path before the first player action, or the scenario definition must explicitly declare and validate a narrowly scoped exemption.
- The initial-breakage gate must verify the declared lock condition and must not accept a fixture that is already open by accident.
- The lock path remains subject to scenario-directory containment and cleanup rules.

Focused verification before resolving this thread:
- Build the feasibility fixture and assert the lock exists and is closed before the player command.
- Exercise the exemption path, if retained, and require explicit metadata plus a separate acceptance result.
- Re-run the initial-breakage gate after reset and confirm it rejects an open/missing/escaped lock.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.