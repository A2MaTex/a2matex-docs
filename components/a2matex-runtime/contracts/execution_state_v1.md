# Execution State v1 Contract

**Status:** Accepted. G16 RAM storage, G17 reads, and G18 live store ownership/Set Variable execution implemented and Linux-verified. Phase H controls remain later work.

**Phase:** A5 — Execution schemas; G16 — scoped variable state extension

## 1. Purpose

Runtime persists enough state to isolate each selected device, survive client
disconnection, expose authoritative progress, and identify uncertain work after
a Runtime crash. The MVP uses three execution records:

```text
RunGroup
└── DeviceExecution
    └── NodeAttempt
```

The original execution records are storage-independent. Phase D maps them to
SQLite and implements transactions; A6 defines referenced errors and events;
A7 defines JSON API projections. Section 9 adds scoped variable state for G16;
its RAM layout and internal interfaces belong to the G16 specification.

## 2. Shared rules

- Runtime-created record IDs are lowercase UUID v4 values.
- Timestamps use the A1 UTC RFC 3339 millisecond format.
- A RunGroup contains at least one DeviceExecution.
- A selected ADB serial occurs at most once in one RunGroup.
- At most one non-terminal RunGroup exists. `RUNNING` and `PAUSED` are
  non-terminal; a second admission fails with the A6 busy error.
- Each DeviceExecution owns its attempts and node outputs; those are not shared
  between siblings. Shared mutable state is available only through explicit
  RunGroup variables under section 9.
- States change only through the transitions listed in this contract.
- Terminal records never return to a non-terminal state.

## 3. RunGroup

A RunGroup is one accepted execution request across one or more selected ADB
serials.

| Field | Type | Rule |
|---|---|---|
| `id` | UUID | Runtime-created identifier |
| `requestId` | UUID | Client-created A7 run idempotency key; unique and immutable |
| `status` | RunGroupStatus | Authoritative group lifecycle state |
| `createdAt` | Timestamp | Set when admission commits |
| `finishedAt` | Timestamp or null | Required only for a terminal state |

`RunGroupStatus` is exactly:

- `RUNNING`
- `PAUSED`
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`
- `INTERRUPTED`

Allowed transitions:

| From | To |
|---|---|
| `RUNNING` | `PAUSED`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `INTERRUPTED` |
| `PAUSED` | `RUNNING`, `CANCELLED`, `INTERRUPTED` |

Admission creates the group directly as `RUNNING`; the MVP has no group-level
`CREATED` or `QUEUED` state. Execution capacity is derived from admitted
membership and has no separate configuration or stored field. Group counters
are derived from DeviceExecution states.

`requestId` prevents a retried HTTP start request from creating another real
execution after the first response was lost. It is not a display number,
execution ordering key, or replacement for the Runtime-created `id`.

## Device selection and execution capacity

One active RunGroup owns a finite, explicit non-empty selection with no fixed
Runtime device-count ceiling. Its execution capacity equals the number of
admitted devices. Users control resource usage by selecting fewer or more
devices; there is no separate maximum-device preference or YAML/env/CLI setting.
Practical capacity depends on hardware. Existing resource/deadline failures
remain failures, never permission to repeat an uncertain device action.

Admission, snapshots, sibling aggregation, cancellation and recovery handle
all accepted/stored members. A new run's smaller selection cannot truncate or
invalidate an older group. The fixed count ceiling is removed in code; H2's
configuration-removal revision derives reservations directly from membership.
H4 exposes complete multi-device dispatch through the existing run API.

## 4. DeviceExecution

A DeviceExecution is one isolated execution of the admitted Runtime IR on one
accepted ADB endpoint.

| Field | Type | Rule |
|---|---|---|
| `id` | UUID | Runtime-created identifier |
| `runGroupId` | UUID | Parent RunGroup |
| `adbSerial` | String | Exact revalidated A3 serial; immutable after admission |
| `status` | DeviceExecutionStatus | Authoritative per-device state |
| `nextNodeId` | String or null | Node to execute at the next safe boundary; null after termination |
| `createdAt` | Timestamp | Set with the admitted RunGroup |
| `startedAt` | Timestamp or null | Set when the scheduler first starts this execution |
| `finishedAt` | Timestamp or null | Required only for a terminal state |
| `terminalError` | A6 error or null | Present for `FAILED` or `INTERRUPTED` when an error applies |

`DeviceExecutionStatus` is exactly:

- `QUEUED`
- `RUNNING`
- `PAUSED`
- `SUCCEEDED`
- `FAILED`
- `CANCELLED`
- `INTERRUPTED`

Allowed transitions:

| From | To |
|---|---|
| `QUEUED` | `RUNNING`, `PAUSED`, `FAILED`, `CANCELLED`, `INTERRUPTED` |
| `RUNNING` | `PAUSED`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `INTERRUPTED` |
| `PAUSED` | `QUEUED`, `CANCELLED`, `INTERRUPTED` |

Pause is a group-level MVP control applied at safe node boundaries. The
[implemented H9–H11 behavior](../h_multi_device_execution/h9_h11_pause_resume.md)
parks the existing live workers, then atomically commits PAUSED for all surviving
members. Resume moves paused devices back to `QUEUED` and the group to `RUNNING`
before granting their next attempt permissions. Workers retain their membership
reservations and RAM; QUEUED does not mean starting another Runner.Run or
replaying completed work. Loss of an ADB endpoint fails only the affected execution
with an A6 device error; the MVP does not wait indefinitely in a specialized
offline-pause state.

Workflow inputs remain scoped in Runtime memory while the process is alive.
They are not A5 persisted fields. A Runtime restart interrupts active work
instead of resuming from potentially sensitive inputs or replaying uncertain
side effects.

## 5. NodeAttempt

A NodeAttempt records one actual invocation attempt for one Runtime IR node.
Every node, including flow and utility nodes, uses the same attempt envelope.

| Field | Type | Rule |
|---|---|---|
| `id` | UUID | Runtime-created identifier |
| `deviceExecutionId` | UUID | Owning DeviceExecution |
| `sequence` | Positive integer | Monotonically increases within the DeviceExecution |
| `nodeId` | String | Runtime IR node ID |
| `attemptNumber` | Integer | Starts at 1 and cannot exceed the node's approved `maxAttempts` |
| `status` | NodeAttemptStatus | Attempt lifecycle state |
| `startedAt` | Timestamp | Set before handler invocation |
| `finishedAt` | Timestamp or null | Required only for a terminal state |
| `output` | Object or null | Validated node output, present only on `SUCCEEDED` |
| `error` | A6 error or null | Present for a failed, timed-out, or interrupted attempt when applicable; absent for cancellation |

The pair `(deviceExecutionId, sequence)` is unique. The pair
`(deviceExecutionId, nodeId, attemptNumber)` is also unique.

`NodeAttemptStatus` is exactly:

- `RUNNING`
- `SUCCEEDED`
- `FAILED`
- `TIMED_OUT`
- `CANCELLED`
- `INTERRUPTED`

Every attempt begins as `RUNNING` and transitions once to one terminal state.
There is no `PENDING` attempt and no `RETRYING` state. A retry creates another
attempt with a higher `attemptNumber` and sequence.

On success, `output` must satisfy the A2 node output contract. Empty-output
nodes store `{}`; Screenshot may store only `artifactId`. Resolved node input,
process stdout/stderr, exit code, and calculated duration are not A5 fields.

## 6. Safe-boundary and checkpoint rules

`nextNodeId` initially contains the admitted Runtime IR entry node. While a
NodeAttempt is running, that attempt identifies the active node and
`nextNodeId` remains unchanged. At completion:

- A retryable failure keeps `nextNodeId` on the same node.
- A completed success or exhausted error branch changes it to the selected
  successor.
- A terminal workflow result changes it to null.

A checkpoint is an atomic persistence operation, not a fourth entity. A node
boundary transaction must:

1. Make the NodeAttempt terminal and store its validated output or A6 error.
2. Update DeviceExecution `nextNodeId`, status, and applicable timestamps.
3. Update the RunGroup state when aggregation changes it.
4. Persist all corresponding A6 events with their Runtime-wide cursors.

The transaction commits before any event is broadcast. Runtime must also
persist a `RUNNING` NodeAttempt before invoking a handler so startup recovery
can identify an uncertain side effect.

G18 publishes a successful Set Variable assignment to
RAM after this checkpoint is confirmed and before readers or execution
continuation are released. Its scalar/map is not a member of the SQLite
transaction. Other outcomes do not publish an assignment. Section 9 defines
the synchronized boundary and the deliberate exclusion of crash resume.

## 7. Group aggregation

Device failures are isolated: unaffected device executions continue. The group
remains `RUNNING` until its remaining devices reach a terminal or paused state.

- `PAUSED`: every non-terminal device execution reached `PAUSED`.
- `SUCCEEDED`: every device execution is `SUCCEEDED`.
- `FAILED`: every device execution is terminal and at least one is `FAILED`.
- `CANCELLED`: an accepted group cancellation has completed.
- `INTERRUPTED`: startup recovery found the group non-terminal.

Counts by device state, duration, last node, and per-device terminal summaries
are derived projections. They are not duplicated RunGroup fields or separate
summary entities.

## 8. Startup recovery

Runtime never automatically resumes an execution after process termination.
In one recovery transaction it changes:

- every non-terminal RunGroup to `INTERRUPTED`;
- every non-terminal DeviceExecution in those groups to `INTERRUPTED`; and
- every `RUNNING` NodeAttempt in those executions to `INTERRUPTED`.

This preserves evidence that a side-effecting handler may have completed before
the process stopped without claiming that it is safe to replay.

## 9. Scoped variable state

**G16–G18 extension — RAM storage, reads, live ownership, and Set Variable
execution implemented and Linux-verified.** Phase H owns actual Pause/Resume
and control integration. Waiter registration/notification and the Wait Variable
node are deferred under H6–H8; H9 proceeds with current executable nodes.
The Set Variable handler is executable; its maps add no
persisted record or separate execution lifecycle.

### 9.1 Ownership, encoding, and bounds

| Scope | Key | Visibility |
|---|---|---|
| `FLOW` | `(deviceExecutionId, name)` | The owning device execution only. |
| `RUN_GROUP` | `(runGroupId, name)` | Device executions admitted to that RunGroup on one Runtime. |

Global is the authoring name for `RUN_GROUP`, not a permanent account-wide,
process-wide, or cross-run store. A new admitted run begins with empty scopes.
Names are case-sensitive and use the existing variable-name contract. First
write creates; subsequent writes replace, including replacement with a
different scalar type. No prior declaration is required.

Values are compact JSON scalars: string, finite `float64` number, or boolean.
Null and containers are unsupported. Missing is separate from present false,
zero, or empty string. Preserve floating-point comparison semantics and reject
invalid UTF-8, invalid/trailing JSON, and non-finite numbers.

| Bound | Value |
|---|---|
| Raw and encoded bytes per value | 64 KiB each |
| Names per Flow or RunGroup scope | 512 |
| Aggregate name plus encoded-value bytes per scope | 1 MiB |

Replacement changes byte accounting by its size delta and does not consume
another name slot. Enforce bounds in the synchronized RAM store before the
successful-checkpoint closure is invoked. The current single-device payload
bound is 2 MiB; for N admitted devices the variable payload bound is
`(N + 1) MiB`, with no fixed device-count ceiling. Map overhead, temporary
copies, inputs, and node outputs are separate from those payload totals.

### 9.2 RAM publication and checkpoint ordering

Keep live variables only in the Runtime process's RAM, owned by the RunGroup.
There are no variable tables, migrations, repositories, shutdown snapshots,
or durable variable history. Existing SQLite execution checkpoints remain
unchanged.
The [G16 storage specification](../g_workflow_engine/g16_scoped_variable_storage.md)
defines the codec, memory quotas, scope gate, and Get/Apply/Close interfaces.

A candidate remains unpublished while G18 invokes its existing successful
node-checkpoint closure under the variable-store gate. After confirmation,
publish the RAM value before releasing readers or continuing execution.
Set Variable's successful node output remains `{}`. The checkpoint and event
payloads do not gain variable names, values, or maps.

If commit confirmation requires D4's existing read-back, retain the candidate
in the live invocation until that result is known. Do not add variable data
to checkpoint requests/read-back. An unresolved persistence failure publishes
nothing and fences the store/execution. If Runtime dies between checkpoint
commit and RAM publication, the old execution is interrupted after restart;
it cannot resume into missing state.

G16 supplies the RAM boundary, G17 supplies binding reads, and G18 owns its
engine/checkpoint integration and failed-candidate finalization. Failed,
timed-out, cancelled, or retried attempts publish no assignment.

Concurrent Global writes use last-committed-write-wins. Individual reads see
published committed state; several reads are not a multi-key snapshot.
Switch followed by Set Variable is not atomic compare-and-set. Phase H adds
cross-device scheduling. Waiter notification after publication remains deferred
under H6–H8; a future waiter stays owned while suspended and must respect
pause/cancel authority. See the [deferral decision](../phases/phase_h.md#wait-variable-deferral--approved-2026-11-22).

### 9.3 Pause, cleanup, restart, and confidentiality

Pause/Resume while Runtime stays alive preserves the same maps and accounting.
An HTTP disconnect does not own the accepted run's variable lifetime. Keep the
group store while sibling executions remain active, then close/clear it after
the group is terminal and workers have joined. Intentional cancellation and
owner shutdown settle work before cleanup. Releasing references makes the
memory eligible for Go garbage collection; immediate RSS reduction is not
promised.

Runtime exit/crash loses variables. Startup recovery marks unfinished work,
including paused runs, INTERRUPTED and restores no maps or execution. New runs
start empty. Replaying an old request ID returns its existing snapshot without
creating another store. Pause/Resume does not mean restart-and-resume.

Flow and Global values are RAM-only for Development and future Production.
They must not be serialized into variable files, SQLite, ordinary logs,
events, Runtime Info, current run snapshots, diagnostic dumps, or exports.
Existing node-output persistence is unchanged; a scalar present in a producer's
ordinary output history is not a variable snapshot or a restoration source.

Phase J still owns Production authorization/decryption. G16 does not authorize
persistence of decrypted workflows, keys, or credentials. There is no separate
plaintext-variable persistence policy to implement for either mode.

## 10. Excluded responsibilities

A5 does not define:

- SQLite DDL, indexes, migration files, or concrete repository implementations;
- general execution-history retention and deletion APIs;
- event envelopes, cursor retention, error codes, or redaction rules;
- HTTP/SSE request and response shapes;
- process diagnostics, logs, or artifact metadata;
- persistent device inventory or device snapshots;
- Production release, license, authorization, or package fields;
- test-one-node or arbitrary run-from-node semantics; or
- automatic device-offline recovery or crash replay.

## 11. Checkout condition

A5 is complete when all execution records, states, transitions, isolation
rules, checkpoint ownership, aggregation rules, and interruption behavior have
one unambiguous owner without selecting a persistence implementation for the
original execution records. G16's variable extension is complete only after
its separate storage acceptance criteria pass; it does not complete G17/G18.
