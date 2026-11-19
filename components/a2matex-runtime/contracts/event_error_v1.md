# Event and Error v1 Contract

**Status:** Accepted

**Phase:** A6 — Event and error schemas

## 1. Purpose

A6 defines durable notifications for committed A5 state changes and one safe
error value shared by execution records, events, and later API envelopes. It
does not define operational logging or SSE wire framing.

```text
A5 state transaction
        -> persisted A6 event
        -> committed Runtime-wide cursor
        -> A7 SSE delivery
```

A5 remains authoritative. Events let a client update its view after a known
cursor; an A7 current-state snapshot repairs an expired or unknown client
view.

## 2. ExecutionEvent

Every persisted execution event contains exactly:

| Field | Type | Rule |
|---|---|---|
| `cursor` | String | Positive decimal, opaque to clients, strictly increasing Runtime-wide |
| `type` | EventType | One of the four closed MVP event types |
| `occurredAt` | Timestamp | A1 UTC RFC 3339 millisecond format |
| `runGroupId` | UUID | Required on every event |
| `deviceExecutionId` | UUID | Required except for RunGroup events |
| `nodeAttemptId` | UUID | Required for attempt and retry events |
| `nodeId` | String | Required for attempt and retry events |
| `payload` | Object | Exact object defined by `type` |

The complete UTF-8 JSON event is limited to 16 KiB. Unknown fields are
rejected. Optional correlation fields are omitted rather than set to null.
There is no separate event UUID, severity, generic message, ADB serial, or
per-device event sequence.

The A6 cursor and A5 `NodeAttempt.sequence` have different purposes:

- `cursor` orders all persisted Runtime events for replay.
- `NodeAttempt.sequence` orders attempts within one DeviceExecution.

## 3. Event types and payloads

The MVP event set is exactly:

### `RUN_GROUP_STATE_CHANGED`

Requires only `runGroupId`. Payload:

```json
{
  "status": "RUNNING"
}
```

`status` must be an A5 RunGroup status.

### `DEVICE_EXECUTION_STATE_CHANGED`

Requires `runGroupId` and `deviceExecutionId`. Payload:

```json
{
  "status": "RUNNING",
  "nextNodeId": "start_01"
}
```

`status` must be an A5 DeviceExecution status. `nextNodeId` is a string or
null and must match the committed A5 record. `error` is allowed only when the
committed DeviceExecution contains a terminal A6 error.

### `NODE_ATTEMPT_STATE_CHANGED`

Requires all correlation fields. Payload:

```json
{
  "status": "SUCCEEDED",
  "attemptNumber": 1,
  "output": {}
}
```

Rules:

- `status` must be an A5 NodeAttempt status.
- `attemptNumber` is 1 through 3 and matches the A5 record.
- `RUNNING` has neither `output` nor `error`.
- `SUCCEEDED` requires the validated A2 output and has no `error`.
- `FAILED`, `TIMED_OUT`, and `INTERRUPTED` require `error` and have no
  `output`.
- `CANCELLED` has neither `output` nor `error`.

### `NODE_RETRY_SCHEDULED`

Requires all correlation fields and refers to the completed failed attempt.
Payload:

```json
{
  "nextAttemptNumber": 2,
  "delayMs": 1000
}
```

`nextAttemptNumber` is 2 through 3. `delayMs` is 0 through 60,000 and must
match the admitted A2 retry policy.

## 4. Persistence, replay, and expiry

Every state-change event is written in the same transaction as its A5 state
change. The event is broadcast only after commit. A cursor is unique and
strictly increasing, but cursors need not be contiguous.

On reconnection, Runtime returns persisted events whose cursor is greater than
the client's last cursor and then continues with live events. A requested
cursor outside retained history produces `EVENT_CURSOR_EXPIRED`.

Recovery from expiry is:

1. Client requests the authoritative A7 current execution snapshot.
2. Runtime returns the current A5 projection and the event cursor included by that
   snapshot's transaction watermark.
3. Client subscribes after that cursor.

This watermark prevents an event from being lost between snapshot retrieval
and SSE subscription. D1/B2 decide retention duration; A7 defines request,
response, `Last-Event-ID`, and SSE framing.

## 5. RuntimeError

Every A6 error contains exactly:

| Field | Type | Rule |
|---|---|---|
| `code` | ErrorCode | Stable closed machine-readable code |
| `message` | String | Sanitized user-facing text from 1 through 500 characters |
| `retryable` | Boolean | Whether this occurrence is proven safe to retry |

The envelope contains no category, boundary, timestamp, correlation ID,
arbitrary details, stack trace, wrapped cause, HTTP status, or raw process
output. Those concerns are either implied by `code`, supplied by the owning A5
record/event, or owned by A7/B4.

## 6. MVP error codes

The closed MVP set is:

| Area | Codes |
|---|---|
| API transport | `INVALID_REQUEST`, `UNAUTHORIZED`, `RESOURCE_NOT_FOUND`, `PAYLOAD_TOO_LARGE` |
| Admission and state | `RUNTIME_BUSY`, `RUNTIME_INCOMPATIBLE`, `VALIDATION_FAILED`, `INVALID_EXECUTION_STATE`, `EVENT_CURSOR_EXPIRED` |
| ADB and device | `ADB_UNAVAILABLE`, `DEVICE_NOT_FOUND`, `DEVICE_UNAUTHORIZED`, `DEVICE_OFFLINE`, `DEVICE_NOT_READY`, `DEVICE_DISCONNECTED` |
| Node and command | `COMMAND_START_FAILED`, `COMMAND_INPUT_UNSUPPORTED`, `COMMAND_FAILED`, `COMMAND_TIMED_OUT`, `NODE_TIMED_OUT`, `NODE_OUTPUT_INVALID` |
| Runtime infrastructure | `PERSISTENCE_FAILED`, `EXECUTION_INTERRUPTED`, `INTERNAL_ERROR` |

Validation diagnostics remain the bounded C1/A7 diagnostic list;
`VALIDATION_FAILED` is only its top-level error code. Cancellation normally
uses the A5 `CANCELLED` state and is not an error.

Connector, observation, OCR, selector, Production authorization, license, and
other deferred feature codes are not reserved in v1.

The four API transport codes are produced only by the A7 boundary before or
while resolving a request. They do not become execution events or persisted
NodeAttempt errors.

## 7. Retryability

`retryable: true` means the producer proved that retry cannot duplicate an
uncertain side effect. It does not instruct Runtime to retry by itself. G8 also
checks the admitted A2 policy, remaining attempts, and lifecycle state.

For Android commands, a failure before process/command dispatch may be marked
retryable. A timeout, ambiguous result, non-zero result after possible
dispatch, invalid output, persistence failure, or interruption is not safely
retryable. The error message never controls this decision.

## 8. Sanitization

An event or error must not contain:

- workflow inputs, credentials, session tokens, or authorization material;
- protected Runtime IR or decrypted package content;
- ADB serials or unrelated device metadata;
- raw stdout/stderr, stack traces, or external-service responses;
- artifact bytes, uncontrolled paths, or arbitrary local filenames; or
- values copied from untrusted input into a message.

Node output is allowed only in `NODE_ATTEMPT_STATE_CHANGED` after A2 output
validation. Screenshot output contains only `artifactId`.

## 9. Logs and excluded notifications

A6 is not a generic logging protocol. B4 owns structured operational logs and
redaction. Process diagnostics stay with E1/B4 and do not enter events.

The MVP has no durable command-started, command-finished, artifact-created,
device-discovery, lifecycle-request, heartbeat, observation, or business-
confirmation event. State-change events already express their committed
effects; A7 owns transient SSE heartbeats.

Publisher-authored `utility.log` event persistence is deferred until its
workflow-input and sensitive-data rules are approved before F8. It is not an
arbitrary A6 payload escape hatch.

## 10. Checkout condition

A6 is complete when Runtime and clients can identify committed state changes,
replay them from one cursor, recover from cursor expiry using a watermarked A5
snapshot, classify failures without parsing messages, and make retry decisions
without exposing protected data.

A8 provides the [portable event and error cases](./contract_fixtures_v1.md)
that later Go and TypeScript tests must decode consistently.
