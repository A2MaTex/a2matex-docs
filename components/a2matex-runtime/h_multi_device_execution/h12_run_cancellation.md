# H12 — Cancel the current run while Runtime stays alive

## Overview

H12 lets the owner cancel the current RunGroup without shutting down Runtime.
It fences new attempts, interrupts the group's active work, joins its workers
and pending controls, settles unfinished SQLite records, then releases the
group's RAM and membership. The owner can start a new workflow afterward.

For Start → Delay → Tap → End, Cancel during Delay interrupts that Delay and
prevents Tap. Previously committed Start results remain. Cancel during Tap
stops waiting for the local command, but cannot undo a touch Android already
received. Runtime records the cancellation without resending the command.

```mermaid
sequenceDiagram
    participant C as Trusted owner caller
    participant L as Existing group lifecycle
    participant W as Device workers and pending Pause/Resume
    participant P as Existing checkpoint service and SQLite
    C->>L: Cancel(runGroupId)
    L->>L: Latch group-only authority and deadline; fence attempts
    L-->>C: ACCEPTED
    L->>W: Cancel group context; invalidate pending control release
    W->>P: Settle any begun attempt under existing bounded rules
    W-->>L: Workers, callbacks and older controls joined
    L->>P: Settle remaining unfinished records for this exact group
    alt Confirmed or already settled by its workers
        P-->>L: Confirmed durable outcome and events
        L->>L: Release RAM and ownership; permit later run
    else Persistence or join failure
        L->>L: Fence Runtime through existing fatal supervision
        Note over L,P: Retain unfinished evidence for D6; never redispatch
    end
    Note over C,L: HTTP actions are H13; Wait Variable H6-H8 remains deferred
```

**Status — H12 implemented and Linux-verified on 2026-11-23.**
Focused H12 tests were approved before addition. Section 14 records actual
verification and platform limits; the public action endpoint remains H13.
H1–H5 and H9–H11 supply the existing foundation. G11 still owns process shutdown;
H12 does not close HTTP, SQLite or the Runtime process. H14/H15 later extend
process and combined acceptance. Further test additions still require scope
agreement under Runtime AGENTS.md.

## 1. Scope and design choice

| Approach | Consequence | Decision |
| --- | --- | --- |
| Reuse `PrepareStop`/`Shutdown` for a Cancel button | Permanently closes the Runtime work gate and drains the process. | Keep for owner exit only. |
| Change SQLite to CANCELLED while workers keep running | A handler, variable publication or artifact callback can outlive its owner. | Excluded. |
| Extend the current group's authority and joined retirement | Reuses cancellation-aware handlers and existing checkpoint rules; Runtime stays available. | H12 implementation. |

The phase adds one internal group action and the minimum lifecycle/checkpoint
extensions needed to make it safe. No new scheduler, database schema, migration,
dependency, device limit, detached cleanup service or durable continuation is
required. Keep service rules outside repositories.

Authoritative references:

- [H1 group ownership](h1_group_execution_ownership.md) and
  [H3 worker completion](h3_worker_completion_and_failure_isolation.md).
- [H9–H11 Pause/Resume](h9_h11_pause_resume.md).
- [G11 owner shutdown](../g_workflow_engine/g11_runner_shutdown.md).
- [A5 states](../contracts/execution_state_v1.md),
  [A6 events/errors](../contracts/event_error_v1.md), and
  [A7 lifecycle actions](../contracts/local_api_v1.md#9-execution-reads-and-lifecycle-control).
- [D4 checkpoints](../d_persistence/d4_atomic_checkpoint.md) and
  [D6 interruption](../d_persistence/d6_startup_recovery.md).

## 2. Existing code and the specific gaps

Paths in this section are relative to `a2matex-runtime`.

| Existing implementation | Reuse | H12 gap |
| --- | --- | --- |
| `internal/services/executionlifecycle/group.go` | Group context, selected membership, live workers, shared variable store and final retirement. | Retain the owner through cancellation settlement and joined controls, rather than retiring as soon as workers return. |
| `internal/services/executionlifecycle/control.go` | Serialized Pause/Resume acceptance, operation identity, boundary fence and drain accounting. | Cancel must supersede a pending control and prohibit its later Resume release. |
| `internal/services/executionlifecycle/lifecycle.go` | Trusted `attempts.Control`, expected cancellation classification and G11 drain. | Cancellation authority must be group-specific; an expected live Cancel must not become a fatal worker error. |
| `internal/execution/attempts/completion.go` | Accepted cancellation normalizes ordinary active outcomes to CANCELLED; infrastructure failures remain failures. | Supply the correct group authority and deadline to each owned attempt. |
| `internal/services/checkpoint_cancellation.go` | Writer gate, residual settlement plan, conditional updates and atomic event batches. | Current request has only a timestamp and discovers the unfinished group globally. Live cancellation needs an exact expected group identity. |
| `internal/execution/runner/runner.go` | Context checks, interruptible retry delay, boundary callback and same-attempt persistence. | Keep all permission checks aligned with accepted Cancel; do not rebuild or restart a runner. |

Before H12, `runRegistered` assigned process-wide `l.control`. Simply calling
`g.cancel()` would leave `GroupCancellation=false` and can classify its expected
context error as fatal. Merely setting the process-wide `accepted` flag would
grant cancellation authority to unrelated/future work. Neither shortcut is valid.

## 3. Internal action contract

Add a lifecycle method equivalent to:

```go
Cancel(ctx context.Context, runGroupID string) (constants.GroupControlDisposition, error)
```

Use the existing `ACCEPTED` and `ALREADY_TARGET` dispositions and existing
classified input/control-state/stopping errors. The exact helper names may
follow project conventions. New enums and sentinel errors, if necessary,
belong in `pkg/constants` and `pkg/x-error`.

Before acceptance validate the caller context, UUID and Runtime work gate.
Under the lifecycle mutex bind the operation to the exact current, fully
registered group and its generation. No caller-supplied cancellation authority,
device list, deadline, node position or callback is accepted.

| Current live state | Internal result and effect |
| --- | --- |
| Runnable or paused group with work remaining | Latch one owned Cancel; return ACCEPTED. |
| Pause pending, or Resume pending | Accept Cancel as higher priority; fence continuation and supersede that operation's authority to release workers. |
| Same group's Cancel already accepted and still settling | ACCEPTED; same operation and deadline, no additional coordinator or events. |
| Same live owner retained briefly after confirmed cancellation | ALREADY_TARGET only after settlement and cleanup are confirmed. |
| Different/missing live group, registration incomplete, or natural completion already won | Control-state error; no fabricated cancellation. |
| Runtime stopping or failed | Existing stopping/failure classification; do not reopen any gate. |
| Context cancelled before the acceptance ordering point | Context error; no side effects. |

After retirement, the lifecycle need not retain a cache of old groups. H13
will use the authoritative stored snapshot to distinguish an already-CANCELLED
target, a different terminal state and a missing record. Repeated cancellation
of an achieved target must ultimately be a no-op; H13 owns its HTTP response.

The request context controls submission only. After ACCEPTED, disconnecting or
cancelling that caller does not undo the operation. The coordinator remains
owned by Runtime and participates in G11 drain accounting.

No persisted CANCELLING state is added. Snapshots can remain RUNNING or PAUSED
while cancellation is pending. Acceptance alone does not mean resources have
been released or Android has undone an action.

## 4. Acceptance ordering and group-only authority

Under the lifecycle mutex, accepting Cancel must:

1. Recheck the exact owner and action eligibility.
2. Latch a monotonic `cancel accepted` state and one operation identity/deadline
   on that group. Further Pause/Resume requests become ineligible.
3. Fence all first/next/retry attempt permissions and stop any remaining group
   queue production. Do not close the Runtime's permanent admission context.
4. Register the cancellation coordinator in existing shutdown drain accounting
   before releasing the mutex.
5. Capture the cancellation signal, unlock, then cancel the group context.

Trusted control snapshots for workers must combine matching-group Cancel with
existing G11 process-stop authority. A worker of a later group must never see
the earlier group's flag or deadline. Preserve the existing direct managed-run
path used by G11 tests. Reject externally injected control callbacks as today.

Once a process stop begins, its earlier deadline may shorten the live operation;
no duplicate action or stop signal extends the original deadline. Owner loss
alone never creates intentional cancellation authority. An already accepted
explicit Cancel remains authorized for its own group, while independent fatal
storage/invariant errors retain precedence.

Use an owner-bound bounded completion context detached from the cancelled group
signal only for joining/settlement. For the MVP, use the existing
`constants.CheckpointTimeout` as the total Cancel completion budget, shortened
by an earlier G11 deadline. Do not reset it per worker, checkpoint or duplicate
request. On budget exhaustion, fence through supervision; do not report clean
cancellation or free resources still in use. No new configuration is introduced.

## 5. What cancellation does at each execution position

| Position when Cancel is accepted | Required result |
| --- | --- |
| Before first attempt permission | Prevent BeginAttempt and handler dispatch; residual settlement cancels the QUEUED device without inventing an attempt. |
| Permission granted but handler not started | Existing context/attempt guards prevent invocation when cancellation is observed; a begun row is settled exactly once. |
| Active Delay | Interrupt its timer through context cancellation; do not wait out the remaining duration as Pause does. |
| Active ADB operation | Cancel the owned process through the existing process runner and join its bounded cleanup. Never retry to discover whether the action happened. |
| Retry delay | Interrupt it; preserve the previous FAILED/TIMED_OUT attempt and retry event, create no next attempt. |
| Parked at a Pause boundary | Wake through group cancellation, without closing the Resume gate as if Resume succeeded. No successor starts. |
| Between nodes, bindings or BeginAttempt | Preserve confirmed prior results; prevent a new action once accepted cancellation is observed by the normal guards. |
| Set Variable or Screenshot completing | Join their synchronous checkpoint/publication/cleanup paths before releasing shared resources. Follow the confirmed-outcome rules below. |
| Device already terminal | Preserve its status, timestamps, output, error, attempts and artifacts. |

The acceptance ordering point and existing permission point define the race.
An operation already permitted can have an external side effect before its
context is observed. Cancellation is best-effort interruption of that work,
not rollback of Android state, crypto transfers or other external effects.

### 5.1 Preserve confirmed results and publication

Do not rewrite an already terminal attempt to CANCELLED. If its successful
checkpoint was frozen/committed before cancellation won the existing attempt
completion decision, retain that result and complete its owned publication.
Otherwise accepted cancellation uses the existing cancelled outcome: no output,
retry, error route or successor; no uncommitted assignment is published and no
uncommitted screenshot is made available. Existing artifact discard rules apply.

A confirmed screenshot and its metadata remain historical user data; do not
delete earlier artifacts when cancelling the group. Flow/Global RAM is closed
after all users finish, including any callback completing a confirmed write.
Values are not copied to SQLite. A later run starts with fresh variable scopes.

## 6. Cancel versus pending Pause/Resume

Cancel may supersede a pending Pause/Resume. Do not make the user wait for a
long Delay to reach the Pause barrier before accepting Cancel.

| Race | Required ordering |
| --- | --- |
| Pause waiting for boundaries | Cancel invalidates that operation and wakes workers. The old coordinator exits; it cannot later commit a stale boundary snapshot. |
| Pause checkpoint already writing | Let its bounded transaction confirm or fail under the writer gate, then join it before residual cancellation. A confirmed PAUSED event may precede CANCELLED. |
| Resume checkpoint already writing | Let it confirm or fail; even if QUEUED/RUNNING commits, the cancellation fence prevents worker release/continuation. Join before final settlement. |
| Resume already released workers before Cancel ordering point | Apply normal active-work cancellation; do not pretend those permissions were never granted. |
| Pause/Resume submitted after accepted Cancel | Reject; no gate reopen, boundary mutation or second action coordinator. |
| Old control wakes after a later group has started | Its captured group/operation identity prevents any effect on the new owner. |

Every release path must check cancellation as well as existing stopped/stale
conditions. Do not only invalidate `g.control` and forget the older goroutine:
its completion and any in-flight transaction remain joined and drain-counted.
Expected supersession is not a new fatal stale-state error. A genuine checkpoint
failure remains fatal even when cancellation is also pending.

## 7. Joining, settlement and retirement

The cancellation coordinator owns completion. Existing worker retirement must
cooperate with it; two finalizers must not independently close RAM or clear
`l.group`.

Required order:

1. Fence and signal the group as described above.
2. Join its worker bodies, handler/process cleanup, attempt callbacks and all
   superseded control operations. A worker returning is not sufficient if its
   reservation-release callback or older control transaction is still running.
3. Using the same CheckpointService writer gate, settle remaining unfinished
   rows for the exact accepted RunGroup. Workers may already have cancelled
   active attempts; residual settlement must not produce duplicate events.
4. Confirm the group's durable terminal result or the legitimate natural
   terminal winner. A successful Cancel with unfinished members yields CANCELLED;
   a completion that already won is preserved.
5. Close the group's shared RAM once, clear continuation/boundary references,
   release membership and current ownership, then signal group completion.
6. Release the cancellation operation's shutdown-drain registration.

Do not wait for `g.done` to join work if the same finalizer is responsible for
closing `g.done`. Use an explicit internal workers/controls-settled signal or
equivalent existing join mechanism. The control coordinator must not wait on
its own drain registration. No blocking wait, cancellation callback, store close
or SQLite operation runs while holding the lifecycle mutex.

A new run cannot dispatch against the old group's RAM or take its ownership
before this retirement completes. D2 can observe a terminal database state
slightly earlier; retain the existing bounded prior-owner handoff join rather
than weakening the one-group ownership rule. A later accepted run uses fresh
workers, inputs and RAM. Starting the old request ID remains replay, not restart.

## 8. Persistence contract and events

Extend `execution.CancellationRequest` with an optional expected RunGroup ID
(exact Go field name follows conventions):

| Caller | Identity requirement |
| --- | --- |
| Existing G11 process shutdown | May retain its store-wide behavior for the exclusively owned database. |
| H12 live Cancel | Must supply and validate the accepted RunGroup ID. |

Inside the existing writer transaction, verify the discovered unfinished group
matches the expected identity **before any write**. If no unfinished group
remains, a bounded read of the exact target must distinguish already completed
cancellation, a natural terminal winner, missing data and corruption. Never
fall through to cancelling another group's records. Reuse existing snapshot
reads or add a narrow repository read; no generic query abstraction is needed.

Do not add a query/update per member. Reuse complete-group reads, service-side
planning, conditional bulk writes and event batches inside one transaction.
Terminal siblings are untouched; only residual non-terminal devices and any
legitimately remaining RUNNING attempts are cancelled. Clear nextNodeId and set
finishedAt on changed devices; preserve startedAt, prior attempts and artifacts.

Event behavior follows existing G9/D4/G11 boundaries:

- Active-attempt FinishAttempt can emit its existing attempt/device/group events.
- Residual settlement emits only the events for rows it changes, using existing
  cancellation-plan ordering: residual attempt IDs sorted, then device IDs sorted,
  then the group-state event. No admission-order rewrite is needed in H12.
- Do not promise one atomic transaction for the entire live cancellation:
  different active attempts settle separately. Each checkpoint is atomic with
  its events; the final residual transaction is atomic with its own events.
- If accepted cancellation remains authoritative when all devices settle, group
  CANCELLED takes precedence over failures among terminal siblings. Preserve those
  siblings' FAILED status and errors in summaries.
- Duplicate actions and no-change residual settlement emit no new events.
- Notify observers only after confirmed commit. No synthetic action entity,
  cancellation-intent event or attempt is created.

Keep confirmed commit handling on existing checkpoint boundaries. The current
residual cancellation method treats commit/persistence errors as fatal; H12 need
not add a new unknown-commit recovery protocol. An unconfirmed commit must never
be retried as a fresh write, reported as clean success or followed by dispatch.
Preserve evidence and use existing failure/D6 behavior.

## 9. Stop, failure and deadlines

| Condition | Required behavior |
| --- | --- |
| Live Cancel completes normally | Runtime HTTP/checkpoints/store remain open; another run is possible. |
| Intentional owner stop during Cancel | G11 closes global admission, shortens the deadline if needed, joins the accepted operation and then performs its existing residual shutdown settlement. It must not close SQLite early. |
| Owner loss during Cancel | No new intentional-stop authority is invented. Already accepted group cancellation may finish within the remaining budget; any unfinished evidence is left for D6. |
| Storage or invariant failure | Keep the original error, fence all new work through existing supervision; no fake successful Cancel or retry. |
| Handler/process/control fails to join within the budget | Fail through supervision; do not close RAM still in use or continue serving new workflow dispatch. |
| Forced process exit | D6 later marks remaining non-terminal records INTERRUPTED; no automatic run resume or command redispatch. |
| All devices naturally complete before Cancel wins | Preserve their final group result. No rewritten history or fabricated cancellation event. |

Cancellation is not a handler failure eligible for retry or an error edge.
Expected context cancellation for the matching accepted group is normal control
flow. Persistence, invariant and cleanup failures must not be hidden by a
concurrent context error. A failure to cancel safely can stop Runtime; keeping
the process usable is the successful-path promise, not permission to ignore errors.

## 10. Logging, security and cost

Use existing contextual error wrapping and the lifecycle/supervisor logging
boundary. Record operation, runGroupId, phase/reason, duration and classified
error on failures. If adding action lifecycle logs, keep one accepted/completed
record per operation; duplicate actions should not flood logs. Do not log raw
workflow inputs, variables, screenshots, tokens or protected content.

Only trusted internal callers gain the action in H12. H13 will require the
existing local bearer token and strict action DTO. No new unauthenticated route,
raw ADB strings, per-device cancellation endpoint or cross-run variable access.

Keep cancellation O(N) in admitted membership, using one group operation and
coalesced signals/joins. No polling SQL loop, database I/O inside a member loop,
new device-count ceiling or busy wait. Keep normal attempt-permission checks
in memory and avoid adding a per-node SQLite read.

## 11. Implementation map and review order

| Area | Expected focused change |
| --- | --- |
| `internal/services/executionlifecycle/control.go` and a responsibility-specific cancellation file | Cancel acceptance, generation/fence rules, joined coordinator and duplicate action behavior. |
| `internal/services/executionlifecycle/group.go` | Coordinate worker completion and cancellation finalization; retain owner until settlement/cleanup. |
| `internal/services/executionlifecycle/lifecycle.go` | Group-aware trusted control, expected live-cancellation handling and process-stop drain integration. |
| `internal/execution/cancellation.go` | Exact-group residual cancellation request while preserving G11 caller semantics. |
| `internal/services/checkpoint_cancellation.go` | Identity guard and exact-target no-change confirmation under existing persistence ownership. |
| `internal/repositories` | Only any narrow identity/read support needed by that service; keep policy in services. |
| `pkg/constants`, `pkg/x-error` | Only required shared constants/errors; reuse existing dispositions and timeout. |
| Existing lifecycle/repository/attempt fixtures | Focused cancellation tests and regression reuse as agreed below. |

Review in this order: authority/identity → attempt and control races → joined
retirement → residual persistence → repeated actions and next-run acceptance.
No constructor/Wire change is expected; if one proves necessary, explain it and
regenerate Wire rather than editing generated code manually.

## 12. Acceptance and test plan

Reuse H9–H11's real SQLite lifecycle fixtures, controllable handlers and existing
G11/attempt/artifact/variable tests. Agree on new H12 tests before adding them.
Prefer deterministic handler/transaction gates to sleep-based race assertions.

| Scenario | Required observable evidence |
| --- | --- |
| Cancel active Delay across two devices | ACCEPTED returns promptly; both stop without waiting full Delay; no successor attempt; residual rows/group CANCELLED; Runtime remains started. |
| Cancel before first BeginAttempt | Zero attempt/handler records for that member; device CANCELLED with startedAt still null. |
| Cancel active Android command | Controlled runner receives cancellation and cleanup is joined; dispatch count remains one. Real-device check reports possible prior side effect honestly. |
| Cancel while retry delay runs | Original terminal attempt/retry evidence unchanged; no retry BeginAttempt. |
| Cancel a PAUSED group | No Resume checkpoint or successor dispatch; cancellation commits and parked workers/RAM are released. |
| Cancel during Pause barrier and Pause/Resume write | Superseded coordinator joins; any committed intermediate events are ordered; Resume never releases work after Cancel fence. |
| Duplicate Cancel and caller disconnect | One operation/deadline/settlement; accepted operation survives submission-context cancellation; no duplicate events. |
| Wrong group and malformed UUID | No context signal, row write, event or effect on the actual current group. |
| Mixed terminal/active members | Earlier success/failure and artifact history survive; only survivors cancelled; final cancellation summary retains all member outcomes. |
| Natural completion race | Already terminal attempts/results remain; completed natural group is not rewritten solely because its finalizer was still live. |
| Set Variable/Screenshot overlap | Publication/discard matches confirmed outcome; no callback uses closed RAM; committed artifacts remain readable. |
| Consecutive run after Cancel | Old group joined and detached once; new run succeeds with fresh RAM and credentials/process unchanged; old request replay dispatches nothing. |
| Residual write rollback or unknown commit | No clean-success claim, no blind second write or new dispatch; failure reaches supervision and unfinished rows remain recoverable. |
| Stop during Cancel and delayed cleanup | G11 waits for operation and workers; SQLite remains open until joined; original/earlier deadline retained; no deadlock or double close. |
| Complete membership above an old count boundary | Reuse available large-group fixtures; no truncated settlement and atomic residual event batches. Fixture count is not a product limit. |

Focused checks should run lifecycle and affected repository/attempt tests under
`-race`, then existing admission/API/process regressions, build and vet. Run the
applicable complete suite and report failures separately. The currently missing
`cmd/agent/testdata/g22_acceptance.py` is a known baseline gap, not an H12 pass.
Linux execution, Windows cross-compilation and native Windows acceptance must
remain separately labeled.

### 12.1 Inspectable manual demonstration after implementation

Extend or reuse the local H9–H11 smoke workflow rather than adding a production
debug endpoint. Provide a readable fixture and driver with a Cancel choice:

1. Start two controlled devices on Start → long Delay → Home → End.
2. Choose Cancel during Delay; inspect the printed group ID and SQLite path.
3. Confirm terminal Start records survive, Delay cancels, Home count is zero,
   both device rows have no nextNodeId, and the group is CANCELLED.
4. Start a second group in the **same running driver process** and confirm it
   succeeds with fresh variables. This demonstrates live Cancel, not G11 exit.
5. Repeat after Pause and verify Cancel wakes the parked workers without Resume.

The final implementation report must give the actual command, fixture path and
read-only queries for its schema. A controlled Home action is not live Android
acceptance. Public Postman cancellation begins with H13, not this demo.

## 13. Completion boundary and deferred work

H12 is complete when internal group Cancel safely handles supported active,
boundary, retry-delay and paused work; checkpoints and controls are joined;
existing history is retained; resources retire once; and another run succeeds
without restarting Runtime. Record actual tests and platform limits here after
implementation. Preparing this specification does not complete H12.

| Later owner | Deferred responsibility |
| --- | --- |
| H13 | Public action DTO/route, auth, exact 200/202/409 snapshots, not-found behavior and shared/Studio/Launcher API examples. |
| H14 | Wider owner-stop/crash/restart acceptance for active/paused/cancelling groups. |
| H15 | Combined current-feature multi-device acceptance on independent Android targets. |
| H6–H8 | Wait registration removal, suspended-wait cancellation and timeout/notification races when the real Wait Variable node returns. |
| Phase I | SSE observation and client lifecycle UI. Existing snapshot/event reads suffice for internal verification. |

No restart-and-resume support, cancellation rollback of external actions,
per-device actions or permanent variable storage is introduced.

## 14. Implementation and verification evidence — 2026-11-23

The implementation extends the existing group finalizer rather than adding a
second finalization coordinator. `Cancel` latches matching-group authority and
one five-second completion budget, fences attempt permissions and signals the
group. A deadline watchdog reports fatal failure even if a worker refuses to
return. An earlier owner-stop deadline shortens that budget.

The finalizer joins workers/reservation-release callbacks and all previously
accepted Pause/Resume controls, settles the exact cancellation target, closes
RAM and then releases group/drain ownership once. A control cannot release a
Resume gate after cancellation is accepted. The existing direct G11 managed-run
path retains process-only authority. No constructor, Wire, dependency, schema,
migration, HTTP route or per-device limit changed.

| Source area | Implemented behavior |
| --- | --- |
| `internal/services/executionlifecycle/cancellation.go` | Internal Cancel, duplicate pending actions, shared deadline/watchdog, residual settlement and bounded cleanup context. |
| Lifecycle `control.go`, `group.go`, `lifecycle.go` | Matching-group trusted attempt authority, expected cancellation classification, joined controls/retirement and shutdown joining. |
| `internal/execution/cancellation.go` | Optional exact target ID, confirmed terminal status and narrow transactional target-read capability. |
| Repository/service `execution_cancellation.go` / `checkpoint_cancellation.go` | Exact target checked under the writer transaction; late cancellation of an old terminal group never touches a newly admitted group. Existing G11 no-ID behavior preserved. |

Focused tests, using real migrated SQLite, cover:

- Active, paused and Pause-pending cancellation; duplicate submission during
  settlement; cancelled caller/wrong group; no successor attempt; closed old RAM
  and successful fresh group under the same live service.
- Cancel during a held Resume write followed by owner stop: no worker release,
  joined control transaction, earlier deadline and cancellation of both members.
- Cancellation before BeginAttempt, with zero fabricated attempts/start times;
  interrupting a 60-second retry gap while preserving its FAILED attempt.
- Residual cancellation rollback fences Runtime without leaking CANCELLED rows;
  an uncooperative worker deadline reports failure without closing still-used RAM.
- Failed siblings remain FAILED while surviving work cancels. A natural terminal
  checkpoint confirmed before Cancel remains SUCCEEDED with no new events.
- Missing target and old-terminal/new-active-group persistence guards; duplicate
  exact-target cancellation changes neither history nor the new group.

Verification executed:

```bash
go test -race ./internal/services/executionlifecycle ./internal/repositories -run 'TestCancelGroup|TestLiveCancellation' -count=3 -timeout=60s
go test -race ./... -count=1
go test -race ./cmd/agent -skip '^TestG22CombinedEngineAcceptance$' -count=1
go vet ./...
make fmt
make build
git diff --check
```

Focused repeated race tests pass. The complete race command passed every package
except the existing G22 test, which cannot open absent
`cmd/agent/testdata/g22_acceptance.py`. The separate available cmd/agent race run
passes, including real managed-agent/H4 process checks. This is not an unqualified
full-suite pass. Windows amd64 agent and affected lifecycle/repository test
binaries cross-compile; native Windows execution is unverified.

The local [H12 smoke guide](../../../../a2matex-runtime/examples/h12/README.md)
reuses the ignored H9–H11 driver with a `cancel` choice. An actual controlled run
was inspected through an independent read-only SQLite connection: old group
CANCELLED with four prior successful Start/Delay attempts and zero Home actions;
new group in the same process SUCCEEDED with ten successful attempts and two
Home actions. The demo has no real ADB side effects. Real Android cancellation,
H13 HTTP action acceptance and wider H14/H15 acceptance remain separate.
