# H9–H11 — Pause checkpoints, coordination and live Resume

## Overview

These three steps let the owner pause a complete RunGroup and later continue
the same live executions. Pause fences new attempts, waits for already permitted
work to settle, then saves PAUSED. Resume saves the transition back to runnable
state before releasing the same workers. Completed actions are never repeated.

For Start → Tap A → Tap B → Tap C → End, a pause requested while Tap B is
running lets B finish and confirm its checkpoint. The worker parks with Tap C
next. Resume continues at C with the same inputs, committed outputs and variables.
Neither a fresh workflow runner nor reconstruction from SQLite is involved.

| Step | One responsibility | Read/implement first |
| --- | --- | --- |
| H9 | Atomic group/device pause and resume checkpoints, guards and events. | Sections 3–5 and the H9 acceptance rows. |
| H10 | Group pause intent, dispatch fencing, safe-boundary barrier and parked-worker ownership. | Sections 2, 6–7 and H10 acceptance. |
| H11 | Confirm resume, release retained workers and continue their preserved position. | Sections 2, 8–9 and H11 acceptance. |

```mermaid
sequenceDiagram
    participant C as Trusted owner control caller
    participant L as H10/H11 group coordinator
    participant W as Existing device workers
    participant P as H9 checkpoint service and SQLite
    C->>L: Pause(groupId)
    L->>L: Fence new attempt permissions; accept one owned operation
    L-->>C: Accepted; group can still be RUNNING
    W->>W: Settle permitted node, publish confirmed output, finish retry delay
    W->>L: Arrive at boundary and park; retain live state
    L->>P: All surviving members ready: atomic PauseGroup
    P-->>L: Confirm PAUSED devices/group and events
    C->>L: Resume(groupId)
    L->>P: Atomic ResumeGroup: PAUSED devices to QUEUED, group to RUNNING
    P-->>L: Confirmed checkpoint
    L->>W: Release the same parked workers
    W->>P: Normal BeginAttempt before next handler
    Note over L,W: Owner stop wakes and joins parked work through G11
    Note over C,L: Public action endpoint is H13; Wait Variable H6-H8 remains deferred
```

**Status — H9, H10 and H11 are implemented and Linux-verified on 2026-11-22.**
The implementation was built in checkpoint and retained-worker layers, with
focused verification at each boundary. See section 13 for commands, observations
and remaining platform/acceptance limits.
H6–H8 remain [deferred](../phases/phase_h.md#wait-variable-deferral--approved-2026-11-22).
H12 adds live cancellation; H13 adds public HTTP actions. Neither is implemented
by these steps. Existing intentional Runtime shutdown must continue to work.

## 1. Current code and design choice

The [current runner](../../../../a2matex-runtime/internal/execution/runner/runner.go)
keeps its inputs, committed outputs, visited nodes, retry identity and next
position inside `Run`. H3 tracks that call until it returns, then releases
ownership and eventually shared RAM. Returning from `Run` merely to pause would
lose continuation state and incorrectly look like worker completion.

The H2 queue is a one-pass admission/reservation structure; every selected member
already has capacity. It is not a general resume queue. A5's PAUSED → QUEUED
transition remains a durable state change, but it does not require calling
`DispatchGroup`, `startGroup`, `queue.take` or `Runner.Run` again.

| Option | Consequence | Decision |
| --- | --- | --- |
| Cancel the active handler to pause | An action may already have happened; replay is unsafe and Delay would need partial-progress semantics. | Excluded. |
| Return and rebuild a runner from SQLite | Inputs/outputs/variables and retry continuation are not durable; would require a different engine/recovery design. | Excluded. |
| Keep the live runner parked between attempts | Preserves current state, uses existing contexts and adds no per-node state persistence. | MVP implementation. |

Retain each worker's membership reservation while parked. It consumes no ADB
process or busy-polling CPU and does not prevent other admitted members from
running. There is no new device limit, capacity setting or dependency.

## 2. Exact MVP pause behavior

Pause is a **group request**, not a per-device request or instantaneous kill.
The following policy is required for both implementation and tests:

| Position when Pause is accepted | Required behavior |
| --- | --- |
| Before a first/next attempt receives permission | Park before bindings, attempt-ID allocation, BeginAttempt and handler invocation. |
| Attempt permission already granted | Let that attempt run once under its normal deadline and finish its existing checkpoint/publication. |
| Active Tap/Swipe/Long Press/Launch/etc. | Do not cancel or resend it. Pause waits for its result and checkpoint. |
| Active Delay node | Let the Delay complete or time out normally; do not freeze or restart its timer. |
| Eligible retry already selected, including an active retry delay | Finish that retry delay, then park before the next retry attempt. The next attempt does not start while pause is pending. |
| Success/error route chosen | Keep the exact selected successor; do not recompute routing on Resume. |
| Set Variable/Screenshot finishing | Wait for checkpoint-confirmed RAM/artifact publication and synchronous cleanup before treating the worker as ready. |
| End/unhandled failure/device failure becomes terminal | Preserve the terminal outcome. Terminal members do not become PAUSED or resume later. |

The dispatch-permission decision under the lifecycle mutex is the ordering
point. Work permitted before the pause fence can enter its handler afterward;
it counts as in-flight work. No attempt may obtain a new permission after the
fence until a confirmed Resume releases it. Tests must distinguish permissions
from the wall-clock moment an external command appears.

For simplicity, Pause does not interrupt the retry timer. Remaining retry delay
is therefore zero when the worker acknowledges its safe boundary; Resume does
not wait the full delay again. A pause request during a long Delay/retry delay
can remain pending for that remaining duration plus settlement. Existing Delay,
retry and node-timeout limits still apply; no new global pause timeout cancels
the action. A client can display pending pause or choose existing owner shutdown.

Between accepting Pause and committing its checkpoint, GET may still show RUNNING
devices with no active attempt. Workers can already be parked in RAM. The MVP
does not persist partial member pauses: one group checkpoint changes all surviving
members together. Do not expose PAUSING, RESUMING or a new NodeAttempt status.

## 3. H9 — Domain inputs, results and repository boundary

Add narrow typed group-control checkpoint contracts under `internal/execution`.
The exact Go names may follow existing conventions; the following data and
checks are mandatory. Use shared constants and `pkg/x-error` for dispositions
and sentinels; do not put enums/errors beside repository methods.

| Contract | Required contents |
| --- | --- |
| PauseGroup/ResumeGroup request | RunGroup ID, one UTC operation timestamp and the coordinator's complete boundary evidence for every non-terminal member. |
| Member boundary evidence | DeviceExecution ID, expected current status, exact nextNodeId, latest confirmed attempt sequence and latest attempt ID or explicit absence before any attempt. No inputs, outputs or variable values. |
| Result | COMMITTED / ALREADY_TARGET / STALE or equivalent typed disposition; group ID, changed member count and committed cursor when a change occurred. Do not fabricate an attempt sequence. |
| Transactional read | Group state, complete ordered member states/next positions, latest attempt identity/sequence per member, and whether any group attempt is RUNNING. Include the event watermark needed for bounded read-back. |
| Repository writes | One conditional bulk member-state update, one conditional group-state update and existing bounded event append batches under one transaction. |

Keep guard logic, evidence comparisons and event construction in the Runtime
checkpoint service. Repositories implement parameterized reads/updates and
affected-row checks. Do not query the database in a per-member loop. Build
evidence maps/events in memory from the complete read, and do not truncate reads
to a page size or current capacity. Terminal members belong to the read but are
excluded from the changed-member set.

Use the existing CheckpointService writer gate and supervisor failure boundary.
Introduce a narrow repository capability for group control and reuse the
existing concrete ExecutionRepository/Tx; do not create an independent writer
service that races D4/G11. If the existing interface is extended, adapt only
affected test doubles. Do not hide a broad interface/constructor refactor in H9.

### 3.1 State-transition table

| Operation | Group before | Changed devices before | Committed target |
| --- | --- | --- | --- |
| Pause | RUNNING | All non-terminal members QUEUED or RUNNING, all at confirmed safe boundaries | Group PAUSED; those devices PAUSED. |
| Resume | PAUSED | Every non-terminal member PAUSED with its matching live continuation | Group RUNNING; those devices QUEUED. |
| Repeat Pause at PAUSED | PAUSED | Already paused | ALREADY_TARGET; no writes/events. |
| Repeat Resume at RUNNING | RUNNING | Already runnable/running | ALREADY_TARGET; no writes/events. |
| Target group terminal, wrong group/evidence or invalid mixed source state | Ineligible | Any | No partial writes; STALE/invalid-state result. |

Group existence/not-found stays distinct from an ineligible transition. Service
input-contract errors stay distinct from storage failure. A missing live owner
cannot be repaired by calling the persistence method to manufacture resume.
H10/H11 must establish ownership before any state-changing checkpoint call.

### 3.2 Required preconditions

For a changing Pause/Resume transaction:

1. Validate group/member identities and the timestamp in memory.
2. Read the group and full member/attempt boundary state once.
3. Require the source group status, at least one non-terminal member and an
   exact match between the surviving member set and coordinator evidence.
4. Require no RUNNING NodeAttempt anywhere in that group. Latest attempt IDs,
   sequences and next positions must match; a zero-attempt member is valid only
   with explicit absence evidence and its unchanged entry position.
5. Require each changed device to have null finishedAt/error and a valid next
   node. Terminal members retain all their existing values and history.
6. Perform conditional updates with exact affected counts. Any mismatch rolls
   back the entire transaction and all its events.

Neither operation inserts or changes node_attempts, artifacts, variable data,
inputs, nextNodeId, startedAt, finishedAt or terminal errors. A never-started
device remains startedAt=null through pause/resume. Already-started devices
retain their original startedAt; existing BeginAttempt uses COALESCE on Resume.
No new SQLite schema, Goose migration, durable continuation or control journal
is required: current status checks/events already allow these transitions.

## 4. H9 — Atomic events, failure handling and idempotence

For M surviving members, emit exactly M device-state events in original
admission order, then one group-state event. Use existing A6 types and payloads:

```json
{"status":"PAUSED","nextNodeId":"tap_c"}
```

Resume uses `{"status":"QUEUED","nextNodeId":"tap_c"}`; the corresponding
group payload is `{"status":"PAUSED"}` or `{"status":"RUNNING"}`. Device and
group events have no nodeAttemptId/nodeId or value/output fields. Reuse existing
event validation, cursor allocation, append batching and wakeup-after-commit.
Do not publish synthetic NODE_ATTEMPT events for pause/resume.

All row changes and M+1 events commit together. Duplicate target-state requests
emit zero events. A later normal BeginAttempt can emit its existing QUEUED →
RUNNING device event; this is separate from the Resume checkpoint's M+1 events.

Bound write/commit/read-back by the existing CheckpointTimeout, shortened by an
accepted shutdown deadline. Do not hold the lifecycle mutex or variable-store
gate through SQLite I/O. Control checkpoints use owner-bound operation lifetime,
not the HTTP caller context after acceptance.

If commit outcome is unknown, retain the dispatch fence and writer serialization
while doing one bounded read-back. Confirm the exact target member set, retained
positions/latest attempts, group status and expected committed event watermark
or event evidence from this operation. No later control transition may start
before this reconciliation ends. A matching read-back acknowledges the original
commit and restores event notification without a second write. An inconclusive
result is fatal persistence uncertainty: use the existing checkpoint/lifecycle
failure path and dispatch nothing to resolve it. Do not blindly retry a toggle.

A confirmed rollback emits no events and cannot release a resume barrier. An
unexpected write/read-back failure after accepted control fences the group and
notifies supervision; do not leave workers running behind a false PAUSED report.
Expected stale/ineligible user actions are safe no-ops, not Runtime crashes.
If captured evidence is unexpectedly stale after an accepted all-worker barrier,
recheck the recorded stop/terminal authority. An expected terminal/stop race is
not a failure; otherwise keep the fence and report an internal control-contract
failure rather than silently reopening dispatch or retrying with guessed data.

## 5. H9 — Implementation and test boundary

H9 delivers service/repository primitives and real-SQLite verification only.
It does not yet accept public Pause requests or park a runner. Tests invoke
the typed methods with explicit valid boundary evidence; production consumers
arrive in H10/H11. Keep tests out of the database adapter's business validation.

Expected source locations, relative to `a2matex-runtime/`:

| Location | H9 change |
| --- | --- |
| `internal/execution/` | Narrow group-control request/result/evidence and transaction capability. |
| `internal/services/checkpoint_*.go` | Pause/resume guards, existing writer-gate integration, events and commit reconciliation. Use focused control-specific files. |
| `internal/repositories/execution_*.go` | Focused control-state read, bulk conditional updates and read-back via existing Tx. |
| `pkg/constants/execution.go`, `pkg/x-error/` | Only necessary typed dispositions/errors following existing placement. |
| Existing checkpoint/repository test fixtures | Reuse admission, history, snapshots, event and injected commit failures. |

H9 is complete when valid/invalid transitions, exact preservation, atomic events,
idempotence, complete above-32 membership and commit-uncertainty behavior are
verified without changing current normal execution behavior.

## 6. H10 — Pause intent and the dispatch barrier

Expose internal group-level `Pause(ctx, groupID)` behavior on the existing
execution lifecycle service, with typed ACCEPTED / ALREADY_TARGET outcomes.
H13 will translate these to HTTP; do not add that route now. Validate caller
context before acceptance. Once accepted, caller cancellation must not revoke
the owned control request, cancel device work or reopen its dispatch gate.

Under the lifecycle mutex, require the matching live group, completed initial
worker registration, open owner lifetime and no opposite pending control.
Set pause intent and capture that group/control generation before returning
ACCEPTED. This is the fence against new attempt permissions. H3 ownership
registration is not itself an attempt permission; control must not mistake an
unregistered/pending initial member for a terminal member.

Add a context-aware boundary callback to runner.Request, separate from the
existing nonblocking attempts.Control cancellation snapshot. It must not couple
the runner package to the lifecycle package or expose handlers to scheduler
state. Without a callback, current direct runner consumers keep their behavior.
Install the callback only inside the trusted group owner. Extend existing
prepared-work/newGroup guards to reject a prebound callback, just as they reject
prebound Control/store authority; the HTTP request and handlers cannot supply it.

The runner calls this boundary once before each new attempt, before mutating
visited/attempt identity or resolving bindings. Reaching it means the previous
iteration's completion, output publication and any selected retry delay are
finished. In ordinary RUNNING mode it grants permission with one small locked
state update and returns immediately; it performs no SQLite read/write.

When pause intent exists, the callback reports immutable boundary metadata and
waits on the group's resume/stop signal. It must not hold the lifecycle mutex
while waiting. An already granted attempt is counted as in-flight until its
worker reaches the next boundary or returns a confirmed terminal result.

Pause completion algorithm:

```text
accept pause intent under lifecycle mutex; prevent new permissions
keep one owner-tracked control operation
wait until initial registration is complete and every surviving worker is parked
    use coalesced notifications, not SQLite polling
    terminal workers count as settled; fatal failure/owner stop wins
if no survivors remain: preserve normal terminal result; write no PAUSED state
capture complete immutable boundary evidence under mutex
release mutex; call H9 PauseGroup through the checkpoint writer
after confirmed commit, publish PAUSED control state under mutex
keep workers parked and owned; finish the tracked control operation
```

Use one coalesced notification for coordinator changes and one group wait gate
per pause/resume cycle. Broadcast to parked workers only when continuation or
stop authority changes. Avoid a goroutine per waiter, timer polling, blocking
notification sends or unbounded channels. A worker's normal terminal callback
must notify a pending pause check so the coordinator cannot miss its completion.

While pause is pending, a device failure remains a device failure. If A fails
and B parks, pause only B and preserve A. If every device becomes terminal, the
derived terminal group result wins; never fabricate a PAUSED group with no
surviving member. A control request accepted just before completion can therefore
observe a terminal result rather than its requested pause target.

## 7. H10 — Ownership, stop and test boundary

Parked workers remain in H3's tracked workers/WaitGroup and retain their existing
group, Flow/Global RAM and reservation. Do not run terminal cleanup, close group
RAM, mark group done or free ownership because all workers happen to be parked.
A PAUSED group remains the one active group and blocks a new admission.

Track accepted control operations in the existing G11 drain accounting (reuse
the lease pattern or an equivalent bounded counter); joining workers alone must
not permit SQLite to close while a control checkpoint/read-back is running.
No accepted operation may outlive its group owner or escape shutdown accounting.

Every parked select and coordinator wait observes the group/owner stop context.
Owner shutdown fences new controls/attempts, wakes parked work and joins workers
and control operations before CancelUnfinished/SQLite close. Reuse intentional
CANCELLED versus forced-exit/D6 INTERRUPTED behavior. Do not implement a public
live CANCEL action as part of H10.

An owner stop racing a successful Pause checkpoint can leave PAUSED rows briefly;
existing intentional settlement then cancels them. A worker must never be
restarted merely to make that cancellation happen. A persistence failure remains
fatal; an expected owner cancellation must not be turned into a spurious fatal
node error. H14 later broadens acceptance, but these stop guarantees are a
minimum requirement of shipping parked workers in H10.

H10 changes focus on the lifecycle group/control implementation, the runner's
boundary callback, and existing group/runner fixtures. Keep the attempt pipeline,
handler interfaces and node-timeout semantics unchanged. It is complete when
new permissions are fenced, in-flight work settles once, the H9 pause checkpoint
is confirmed before acknowledging PAUSED, and parked groups stop cleanly.

## 8. H11 — Resume the same live workers

Internal `Resume(ctx, groupID)` requires the matching live group and preserved
parked continuations. It accepts one owner-tracked operation under the same
control serialization as Pause. Do not reconstruct a group from a saved PAUSED
record; Runtime restart already interrupts that record and discards its RAM.

Resume ordering is mandatory:

1. Keep the wait gate closed and capture the complete surviving member evidence.
2. Call H9 ResumeGroup: devices PAUSED → QUEUED, group PAUSED → RUNNING,
   preserving positions and terminal siblings, with atomic events.
3. After confirmation, recheck owner-stop/failure authority under the lifecycle
   mutex. If stopping, release no new permission; G11 settles the queued rows.
4. Otherwise install the resumed generation and wake retained workers exactly
   once. Do not call queue.take/DispatchGroup or start another Runner.Run.
5. Each worker continues its existing loop, with expected durable device status
   QUEUED for its first post-resume BeginAttempt. After that begin, normal
   RUNNING expectations apply. Existing StartDevice preserves startedAt.

An unexpected H9 ALREADY_TARGET/STALE result when the coordinator believes this
is a new state-changing operation must not authorize wakeup. Resolve expected
stop/terminal or original-commit read-back evidence first; unexplained RAM/SQLite
disagreement is a control-contract failure. Duplicates normally short-circuit
through the coordinator's target/pending-state table, not a second toggle write.

The boundary callback needs to tell the runner whether this permission follows
a confirmed Resume, so its next BeginAttempt does not incorrectly expect
RUNNING. It must not overwrite retry identity, next node or visited-node rules.
Terminal members never receive a resume permission.

The preserved per-device continuation includes:

| Data | Required behavior |
| --- | --- |
| Next node and selected route | Continue the committed successor/error route; do not run the previous handler again. |
| Retry number, previous attempt ID and exhausted delay | Keep next attemptNumber and predecessor; sequence grows normally. No extra NODE_RETRY_SCHEDULED event or repeated delay. |
| Visited nodes and committed output map | Retain them; do not trigger false cycle detection or lose NODE_OUTPUT bindings. |
| Workflow inputs and variable views/store | Same copies and same group RAM; no reseeding or store replacement. |
| Finished attempts and artifacts | Preserve identities, sequences, outputs, metadata and files byte-for-byte. |

No node timeout runs while parked: parking happens outside an invocation. Each
new attempt receives its ordinary full node timeout when the invoker starts it.
Already finished Delay nodes are not resumed as partial timers. Global writes
from other still-running members during pending pause retain their normal
confirmed-publication order; PAUSED means all survivors have reached the barrier.

H11 must support repeated pause/resume cycles without reused closed channels,
duplicate workers or stale generation signals. It is complete when continued
positions/inputs/outputs/variables/retry identity are preserved and actual
commands/attempts are never duplicated.

## 9. Internal action outcomes and client boundary

Keep internal control admission separate from target-state completion:

| Current situation | Internal outcome / effect |
| --- | --- |
| RUNNING, new Pause | ACCEPTED; one fence and operation, even if completion is fast. |
| Pause already pending, duplicate Pause | ACCEPTED; same work, no extra checkpoint/task. |
| PAUSED, Pause | ALREADY_TARGET; no writes/events. |
| PAUSED, new Resume | ACCEPTED; checkpoint first, then wakeup. |
| Resume already pending, duplicate Resume | ACCEPTED; no second checkpoint/wakeup. |
| RUNNING, Resume with no pending opposite action | ALREADY_TARGET; no writes/events. |
| Opposite action pending, initial handoff incomplete or terminal group | Ineligible transition; no queued toggle or partial effect. |
| Invalid/missing group ID, stopped owner or infrastructure failure | Existing typed validation/not-found/stopping/failure handling. |

Use the existing group owner and one small RAM control generation to reject stale
wakeups/callbacks. No action requestId, per-device endpoint, control queue or
persisted intent is introduced. Target-state idempotence does not mean replaying
an old request body can undo a later user action.

Before H13, there is no supported HTTP Pause/Resume endpoint. GET snapshots still
work. During a pending action they can show the preceding committed state; after
pause commit, paused=M and already-terminal counts remain unchanged. Immediately
after resume commit, queued=M is valid; workers may already be RUNNING by the
time GET returns. New run POST remains busy while the group is PAUSED.

Resume does not recompile or re-admit the workflow, replace its selection/inputs,
or add a fresh all-device preflight. If a device was lost while PAUSED, its next
ADB operation follows H5; Delay/RAM-only work has the same observation limits as
before. No paused-device connectivity watcher is introduced.

H13 maps ACCEPTED to 202, ALREADY_TARGET to 200 and ineligible transition to the
existing 409 error+snapshot contract. HTTP request loss does not revoke accepted
control. Do not publish those endpoint examples as implemented in H9–H11.
Future Production authorization may additionally fence Resume under Phase J;
this spec does not bypass or implement that authorization.

## 10. Implementation order and review scope

Implement and verify the three steps separately, using this one file:

| Delivery | Production focus | Meaningful verification before proceeding |
| --- | --- | --- |
| H9 | Typed boundary evidence; checkpoint service; bulk repository transitions and events. | SQLite state/evidence guards, rollback, idempotence and read-back. Normal execution regression passes. |
| H10 | Group pause admission/fence/barrier; runner callback; owner stop/join for parked work. | No new permissions, correct settlement, terminal races and shutdown. Resume remains H11. |
| H11 | Resume checkpoint ordering, generation/wakeup and runner expected-status handoff. | Actual continuation without redispatch, retry identity/RAM preservation and repeated cycles. |

Likely files are the existing `internal/services/executionlifecycle/{group,lifecycle}.go`,
a focused control file in that package, `internal/execution/runner/{runner,state}.go`
and H9's checkpoint/repository contracts. Prefer small cohesive helpers in their
current responsibility folders. Keep DI at the composition root and regenerate
Wire only if constructor wiring actually changes. No direct SQL in runners or
lifecycle coordination, and no pause condition copied into every node handler.

Before a delivery exceeds the roadmap's review thresholds, explain and narrow
the dependency rather than silently implementing all three phases together.
No schema/catalog/backend edit, dependency, process component, configuration
knob, Runtime Info node capability or public API is part of this implementation.

## 11. Proposed acceptance and regression matrix

The user approved focused H9–H11 tests before they were added. The table below
remains the acceptance matrix; section 13 records actual evidence rather than
implying that every row was separately exercised end to end. The earlier H5
existing-tests-only choice was not applied to these approved tests. Reuse real-SQLite checkpoint, H3 group and H4 API/process
fixtures, injected command outcomes and deterministic barriers/clock controls.

| Step | Scenario | Required observation |
| --- | --- | --- |
| H9 | Pause queued and previously started boundary members with terminal siblings | Exact state changes, preserved fields, M+1 ordered events, no attempts added. |
| H9 | Resume with next node/retry predecessor retained | PAUSED → QUEUED, group RUNNING; original IDs/startedAt/next positions and history preserved. |
| H9 | Duplicate target transition | No changed rows, events or cursor advance. |
| H9 | Active attempt, missing/later mismatched member, stale sequence/next position | Whole operation rejected with zero partial changes. |
| H9 | Above-32 membership | Complete reads and bounded append batches in one transaction; no count ceiling. |
| H9 | Failure at member/group/event writes; unknown commit with matching/mismatching read-back | Rollback atomicity or one confirmed original commit; no repeated transition or false wakeup. |
| H10 | Pause before first attempt and race against permission acquisition | No attempt without a prior permit; work already permitted settles once. |
| H10 | Pause during real Delay and during retry delay | Pending until those waits finish; no timer reset or next attempt; no shortened node deadline. |
| H10 | Set Variable/Screenshot at the pause boundary | Publication/cleanup finishes before readiness; committed RAM/artifact stays usable. |
| H10 | One member fails/finishes while siblings park; all members finish during pending Pause | Only survivors pause, or terminal result wins; no terminal member rewritten. |
| H10 | HTTP/caller cancellation after acceptance; duplicate Pause | Same owned operation continues; one barrier/checkpoint, no caller-owned cancellation. |
| H10 | Owner stop/fatal failure with parked members or control write in progress | Every worker/task joins; no deadlock, orphan process or storage close before settlement. |
| H11 | Compile/start → pause → resume → terminal | Original group/member identities, complete history and per-device input pairing; no repeated command. |
| H11 | Retry waiting at pause then Resume | Next attempt number/predecessor/sequence correct; no duplicate retry event or full delay restart. |
| H11 | NODE_OUTPUT and Flow/Global consumers after Resume | Same committed values, no new store and no failure from reset visited state. |
| H11 | Several cycles, concurrent duplicate actions and opposite pending actions | Exactly one transition per accepted cycle; stale gates cannot dispatch; opposite action rejected. |
| H11 | Old-group callback races a later new admission | Captured group identity/generation prevents any control or wakeup from reaching the new group. |
| H11 | Resume write failure/unknown commit/owner stop before release | Zero new handler dispatch until confirmed and authorized; normal failure/stop settlement. |
| All | PAUSED admission/replay, intentional stop, crash/restart | New intent stays busy, request-ID replay never runs again, shutdown cancels and restart interrupts without resume. |

Run phase-focused tests first, existing normal execution/API/managed-process
regressions, available race tests, formatting/build/vet, Windows amd64 cross-build
and diff/link checks. Do not silently skip the known missing G22 Python harness;
report it separately. Distinguish controlled acceptance, real Android and native
Windows evidence. Spec-writing did not execute these checks; implementation evidence is recorded
separately in section 13.

## 12. Observable demonstration and completion gates

H9 can be inspected with real-SQLite service fixtures before a runner integration
exists. H10 can demonstrate stable PAUSED plus intentional owner stop. H11 adds
live continuation. Do not require the not-yet-built H13 endpoint to verify them.

At H11, provide a small readable internal smoke driver following the existing
`test_demo/` pattern and a committed `examples/h9-h11/README.md`/graph fixture.
It calls normal internal start/Pause/Resume methods, not a debug HTTP route.
Keep its SQLite in an explicitly printed, inspectable state directory; never
silently clear the user's main execution database. Keep constructor setup shared
with existing fixtures where practical; the driver is not a second engine.

Use Start → Delay 1 (5000 ms) → Press Home → Delay 2 (5000 ms) → End on two
controlled endpoints, or selected real devices when available. Keep node timeouts
large enough to finish each delay. The driver waits until both first Delay
attempts have begun, requests Pause once, prints pending progress, then waits
for PAUSED. It pauses for user input before Resume and prints IDs/database path.
Controlled commands must be labelled as such and counted; real commands require
the user's selected serials and configured ADB executable.

| Observation point | Expected data |
| --- | --- |
| Fully paused before Press Home | One PAUSED group, two PAUSED devices, nextNodeId=home; four successful attempts total (Start and Delay 1 per device), no RUNNING attempt and no Home dispatch. |
| While the user inspects PAUSED | Attempts, outputs and command counts remain stable; group/member finishedAt stays null, and no new run is admitted. |
| Resume checkpoint alone | Two QUEUED devices, group RUNNING, same next positions and four attempts. A live GET may race the subsequent BeginAttempt and already show progress. |
| Final success | Same group/member IDs; ten successful attempts total, one Home action per member, no repeated Start/Delay 1; final group SUCCEEDED. |
| One pause/resume cycle with two survivors | Three events from each control transaction, six control events total. Normal resumed BeginAttempt emits its separate device RUNNING event. Do not assert a guessed global cursor or count only control events as all events. |

Also demonstrate Stop and exit while PAUSED: no additional action dispatch,
unfinished members settle CANCELLED, historical records survive and the process
exits. A forced-exit test must recover the group as INTERRUPTED, never runnable.
Public Postman lifecycle testing and Studio/Launcher UI examples arrive in H13.

H9–H11 are complete only with their respective evidence recorded here. Wait
Variable and watcher behavior remain deferred; H12–H15 retain their separate
live-cancel, HTTP integration and broader acceptance ownership. Pausing keeps
Runtime alive and retains RAM; shutting down frees it and does not support
restart-and-resume.

## 13. Implementation and verification evidence — 2026-11-22

| Step | Delivered code and evidence |
| --- | --- |
| H9 | `internal/execution/group_control.go`, `internal/repositories/execution_control.go` and `internal/services/checkpoint_control.go`: optional narrow repository capability, complete boundary evidence, bulk updates, existing checkpoint writer gate, batched atomic events and unknown-commit read-back. Focused real-SQLite tests pass for queued/started members, terminal siblings, duplicate cycles, invalid/active evidence, rollback and 128-member transactions. |
| H10 | `internal/services/executionlifecycle/control.go` plus group/lifecycle integration: owner-owned pause intent, constant-time ready/live-count barrier checks, coalesced notification and parked ownership. Existing runner dispatch receives a trusted boundary callback; prepared-work guards reject prebound callback authority. Caller cancellation does not revoke accepted pause. Mixed device failure and intentional shutdown tests pass. |
| H11 | Confirm Resume before closing a fresh group wait gate, retain the same worker and restore its QUEUED expectation. Tests preserve NODE_OUTPUT bindings, separate Flow values, Global RAM, attempt sequence/retry number/predecessor and two pause/resume cycles. Resume rollback never wakes workers; owner stop joins a blocked Resume transaction without dispatching a successor. |

No schema migration, dependency, Wire constructor, handler contract or HTTP
route was added. Shared constants/errors stay in their existing folders. The
checkpoint service and lifecycle supervision own contextual failures; no new
per-node duplicate logging boundary is introduced.

Commands/results:

- Focused H9 real-SQLite tests: `go test ./internal/repositories -run '^TestGroupControl' -count=1` — pass.
- Focused coordination/failure tests under `-race`, repeated three times — pass.
- `go test -race ./... -skip '^TestG22CombinedEngineAcceptance$' -count=1` — available suite passes, including unchanged API/actual-agent regressions. The explicit exclusion is the missing G22 Python harness, not a silently waived failure.
- `go test ./... -count=1` — all packages except cmd/agent pass; existing G22 acceptance still fails because cmd/agent/testdata/g22_acceptance.py is absent. H4 agent acceptance passes.
- `make fmt`, `make build` and `go vet ./...` — pass.
- Windows amd64 agent and repository/lifecycle test cross-builds — pass. Native Windows execution has not been performed.

The [readable smoke walkthrough](../../../../a2matex-runtime/examples/h9-h11/README.md)
compiles/admit-checks the committed graph and runs the real internal runner,
checkpoints and SQLite against two controlled endpoints. It uses a separate,
printed persistent state directory. The local driver is under the intentionally
ignored test_demo directory; the automated regression tests and example graph
are normal repository files.

Observed smoke results, confirmed by read-only SQLite queries:

| Scenario | Observed result |
| --- | --- |
| Pause during both five-second Delays | Group and both devices PAUSED, nextNodeId=home, four successful attempts and zero Home actions; values remain stable during manual inspection. |
| Press Enter to Resume | Same group/device IDs finish SUCCEEDED, ten successful attempts, two Home actions and 44 events scoped to that group. No repeated Start/Delay 1. |
| Stop/EOF while PAUSED | Owner shutdown commits CANCELLED for unfinished work and preserves the four completed attempts. |

Controlled Home actions are not real ADB dispatch. Native Windows, independent
Android-device pause/resume and broader H14/H15 combined fault acceptance remain
separate. Existing artifact/Set Variable checkpoint regressions pass, but there
is no new screenshot-specific pause acceptance scenario in this change. These
limits do not imply Wait Variable support or a public lifecycle-action endpoint;
H6–H8 remain deferred and H12/H13 remain unimplemented.
