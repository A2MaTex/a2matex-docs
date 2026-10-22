# Execution State v1 Contract

**Status:** Accepted

**Phase:** A5 — Execution schemas

## 1. Purpose

Runtime persists enough state to isolate each selected device, survive client
disconnection, expose authoritative progress, and identify uncertain work after
a Runtime crash. The MVP uses three execution records:

```text
RunGroup
└── DeviceExecution
    └── NodeAttempt
```

This contract is storage-independent. Phase D maps it to SQLite and implements
transactions; A6 defines referenced errors and events; A7 defines JSON API
projections.

## 2. Shared rules

- Runtime-created record IDs are lowercase UUID v4 values.
- Timestamps use the A1 UTC RFC 3339 millisecond format.
- A RunGroup contains at least one DeviceExecution.
- A selected ADB serial occurs at most once in one RunGroup.
- At most one non-terminal RunGroup exists. `RUNNING` and `PAUSED` are
  non-terminal; a second admission fails with the A6 busy error.
- Each DeviceExecution owns its attempts and node outputs. Data is never shared
  between sibling device executions.
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
`CREATED` or `QUEUED` state. Concurrency limits and group counters are not
stored fields. Configuration owns the limit, and counters are derived from
DeviceExecution states.

`requestId` prevents a retried HTTP start request from creating another real
execution after the first response was lost. It is not a display number,
execution ordering key, or replacement for the Runtime-created `id`.

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

Pause is a group-level MVP control applied at safe node boundaries. Resume
moves paused device executions back to `QUEUED` so H2 can reapply the current
concurrency limit. Loss of an ADB endpoint fails only the affected execution
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

## 9. Excluded responsibilities

A5 does not define:

- SQLite tables, indexes, migrations, retention, or repository interfaces;
- event envelopes, cursor retention, error codes, or redaction rules;
- HTTP/SSE request and response shapes;
- process diagnostics, logs, or artifact metadata;
- persistent device inventory or device snapshots;
- mutable workflow variables;
- Production release, license, authorization, or package fields;
- test-one-node or arbitrary run-from-node semantics; or
- automatic device-offline recovery or crash replay.

## 10. Checkout condition

A5 is complete when all execution records, states, transitions, isolation
rules, checkpoint ownership, aggregation rules, and interruption behavior have
one unambiguous owner without selecting a persistence implementation.
