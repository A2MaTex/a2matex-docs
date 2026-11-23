# Local API v1 Contract

**Status:** Accepted

**Phase:** A7 — Local API schema

## 1. Purpose and responsibility

A7 defines the authenticated HTTP/JSON and Server-Sent Events boundary used by
Studio to validate, compile, and execute a complete draft workflow. It projects
the accepted A2–A6 contracts; it does not create another workflow, device,
state, event, or error model.

```text
Studio native process
        -> loopback HTTP/1.1 + JSON
        -> Runtime services
        -> committed A5 state and A6 events
        -> authenticated SSE replay
```

The first API is a Development-mode API. Launcher Production package,
authorization, license, and protected-workflow endpoints are added only with
I2 under
[Production Execution Authorization v1](../../../specifications/production_execution_authorization_v1.md).
A successful API response never replaces G1 execution admission.

## 2. Common transport rules

- The base path is `/api/v1` on the configured loopback listener.
- JSON field names use lower camel case. Unknown body fields and unknown query
  parameters are rejected.
- JSON requests use `Content-Type: application/json`; JSON responses use
  `application/json; charset=utf-8`.
- Timestamps use the A1 UTC RFC 3339 millisecond format.
- UUIDs and other identifiers retain their A1/A2/A5 formats.
- Success bodies are direct typed objects. They are not wrapped in a generic
  `data` object.
- Every non-health response includes `Cache-Control: no-store`.
- Runtime does not enable wildcard CORS. Studio calls the API through its
  trusted native process or native bridge, not arbitrary page JavaScript.
- Request bodies are read through an endpoint-specific size limit before full
  decoding. Invalid JSON, duplicate JSON object keys, trailing JSON values,
  unknown fields, and invalid UTF-8 are rejected.

Only `GET /health` is unauthenticated. Every other endpoint requires:

```http
Authorization: Bearer <local-session-token>
```

The token is never accepted in a URL, query parameter, cookie, request body,
event, or log. B5 generates a cryptographically random token at each Runtime
startup. In standalone development it publishes the API base URL and token
through a per-user Runtime connection file restricted to the current user.
The file is written atomically and removed on clean shutdown. This mechanism
does not claim to protect a user from another process already running with the
same fully compromised OS account.

For an approved UI-managed launch, the child hands the API base URL and fresh
session token to its owning native app on the private control channel after
startup recovery and listener readiness. The owner requests process shutdown
on that same channel. HTTP run actions apply to workflow state. The standalone
development handoff uses the protected connection file described below.
See [B3 ownership](../runtime_process_foundation/b3_supervisor.md#ui-managed-runtime-for-mvp)
and [B5 managed credentials](../runtime_process_foundation/b5_local_authentication.md#managed-launch-credential-handoff).

The connection file is a strict `runtime-connection.json` object:

```json
{
  "apiBaseUrl": "http://127.0.0.1:8765/api/v1",
  "sessionToken": "<43-character base64url token>"
}
```

The token contains 32 random bytes encoded as unpadded base64url. Its
OS-specific application-data directory is selected by B2/B5; it is not placed
inside a project or workflow directory.

## 3. Endpoint set

The Development MVP exposes exactly:

| Method and path | Responsibility |
|---|---|
| `GET /api/v1/health` | Minimal unauthenticated process liveness |
| `GET /api/v1/runtime-info` | Authenticated compatibility handshake |
| `POST /api/v1/workflow-validation` | Content-only Editor Graph validation |
| `POST /api/v1/workflow-compilation` | Content-only Editor Graph and Input Schema compilation |
| `GET /api/v1/devices` | Read the current in-memory inventory |
| `POST /api/v1/devices/refresh` | Refresh and return the inventory |
| `POST /api/v1/runs` | Idempotently admit one complete draft run |
| `GET /api/v1/runs/active` | Read the non-terminal run, if one exists |
| `GET /api/v1/runs/{runGroupId}` | Read one authoritative execution snapshot |
| `GET /api/v1/device-executions/{deviceExecutionId}/attempts` | Read paged attempt history |
| `POST /api/v1/runs/{runGroupId}/actions` | Pause, resume, or cancel a group |
| `GET /api/v1/events` | Replay and stream A6 events with SSE |
| `GET /api/v1/artifacts/{artifactId}` | Read one authorized screenshot artifact |

There is no generic CRUD API for execution records. Clients cannot create,
edit, or delete RunGroups, DeviceExecutions, NodeAttempts, events, or artifact
metadata directly.

## 4. Error and status rules

The base JSON failure envelope is:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "request body is invalid",
    "retryable": false
  }
}
```

`error` is the exact A6 `RuntimeError`. A run-admission failure may additionally
contain `validation`, and an invalid lifecycle transition additionally contains
`snapshot`; no other ad hoc error-detail field is allowed. A7 adds four
transport codes required before a domain service can be invoked:
`INVALID_REQUEST`, `UNAUTHORIZED`, `RESOURCE_NOT_FOUND`, and
`PAYLOAD_TOO_LARGE`.

| HTTP status | Use |
|---|---|
| `200` | Read, validation, idempotent replay, or already-achieved action |
| `201` | A new RunGroup was admitted |
| `202` | A lifecycle action was accepted but is not yet reflected by a terminal or paused state |
| `400` | Invalid JSON, fields, path/query values, cursor, or action |
| `401` | Missing or invalid local session token |
| `404` | Requested run, execution, artifact, or route does not exist |
| `409` | `RUNTIME_BUSY` or invalid lifecycle state |
| `410` | Requested SSE cursor is outside retained history |
| `413` | Endpoint payload limit exceeded |
| `422` | Well-formed workflow or run request failed semantic admission |
| `500` | Unexpected internal or persistence failure |
| `503` | Required local subsystem, such as ADB, is unavailable |

The HTTP status communicates transport outcome; client behavior is controlled
by the stable error code, never by parsing `message`.

## 5. Liveness and compatibility

### `GET /health`

Returns only:

```json
{
  "status": "UP"
}
```

It proves that the process can answer HTTP. It exposes no version, mode,
handler, device, execution, build, or host data.

### `GET /runtime-info`

This authenticated response is the v1 compatibility handshake:

```json
{
  "runtimeVersion": "0.1.0",
  "runtimeIrFormatVersions": [1],
  "supportedNodes": [
    {
      "type": "android.tap",
      "runtimeHandler": "android.tap"
    }
  ]
}
```

The `/api/v1` path is the API-version contract, so the body does not repeat an
`apiVersion` field. Development and Production managed clients use this
endpoint. The response omits a fixed mode. `supportedNodes` contains unique A2 type/handler pairs
sorted by `type`. It does not expose handler versions, mutable catalog records,
Go version, commit, build time, operating system, architecture, local paths, or
configuration. Studio may use this response for early feedback, but G1 still
validates the exact Runtime IR on every start request.

### Managed request readiness

Before every Runtime API operation, including opening or reconnecting an SSE
stream, the native client checks the owned child and control channel and sends
a bounded authenticated `GET /api/v1/runtime-info` probe. The probe itself is
sent directly using the credentials from the private ready message. A valid
authenticated response confirms readiness for that operation. Compatibility
checks use the response separately.

If readiness fails before the operation is sent, follow the B3 replacement
policy and verify the new session before sending it. If transport fails after
sending, the outcome may be unknown. Validation and compilation may be
retried after readiness is restored. A run-start retry must reuse the original
request ID and payload. Workflow diagnostics and ordinary operation error
responses leave the Runtime session in place.

## 6. Local source validation and compilation

### `POST /workflow-validation`

Development mode only. The strict request contains exactly:

```json
{
  "editorGraph": {}
}
```

The graph limit is 1 MiB. Validation completion returns HTTP `200` even when
the graph is invalid:

```json
{
  "status": "INVALID",
  "errors": [
    {
      "code": "EDGE_TARGET_NODE_NOT_FOUND",
      "message": "Edge target node does not exist",
      "nodeId": "missing_node",
      "edgeId": "edge_01",
      "path": ["edges", 0, "targetNodeId"]
    }
  ],
  "warnings": [],
  "truncated": false
}
```

The result deliberately matches the approved Backend validation shape:

- `status` is `VALID`, `VALID_WITH_WARNINGS`, or `INVALID`.
- `errors` and `warnings` contain at most 100 diagnostics in deterministic
  order; the combined total is capped at 100.
- A diagnostic contains `code` and safe `message`, plus optional `nodeId`,
  `edgeId`, and `path`.
- `path` contains only bounded string property names and non-negative array
  indexes. It never contains workflow values.
- `truncated` is true when further diagnostics were omitted.

C1 owns the closed diagnostic-code set and ordering. This endpoint performs no
save, compilation, run creation, ADB discovery, or command execution.

### `POST /workflow-compilation`

Development mode only. The strict request contains exactly:

```json
{
  "editorGraph": {},
  "inputSchema": {}
}
```

`editorGraph` is the same authoring document accepted by the validation route.
`inputSchema` uses the A2 reduced schema. The request contains no Project Draft
ID, expected version, Node Definitions, Runtime IR, device, or execution input.
The complete request is limited to 2 MiB, while the Editor Graph retains its
1 MiB limit.

Completed validation and compilation return HTTP `200`. A successful response
contains the complete A2 document:

```json
{
  "validation": {
    "status": "VALID",
    "errors": [],
    "warnings": [],
    "truncated": false
  },
  "runtimeIr": {}
}
```

An invalid source response uses the same bounded diagnostic contract and emits
no partial IR:

```json
{
  "validation": {
    "status": "INVALID",
    "errors": [],
    "warnings": [],
    "truncated": false
  },
  "runtimeIr": null
}
```

Rules:

- `runtimeIr` is non-null only for `VALID` or `VALID_WITH_WARNINGS` after the
  generated document also passes strict A2 validation.
- Runtime resolves definitions and defaults from its immutable installed C7
  compilation registry. The client cannot supply trusted definitions.
- Binding kinds and Runtime IR input sources use the accepted uppercase enum
  values.
- Missing node timeout and retry fields resolve according to A2; no compiler
  default is written back into the Editor Graph.
- The call is deterministic for the same content and installed Runtime build.
  It performs no Project Draft write, Runtime SQLite write, run creation, ADB
  operation, queue publication, or device command.
- Studio may cache the returned IR in its own local Project Draft. G1 still
  validates that exact IR and execution inputs when a run is requested.

Malformed transport input returns the normal A7 `400`/`413` failure rather
than a compilation response. An unexpected compiler failure returns `500` and
must not expose a partial IR or protected source content.

## 7. Session device inventory

These routes are shared by Studio and Launcher when each owns its Runtime
session. See [E7](../e_device_subsystem/e7_device_api.md) for the lifecycle,
error mapping, acceptance cases, and verification status.

Both device endpoints return:

```json
{
  "devices": [
    {
      "adbSerial": "192.0.2.10:5555",
      "status": "UNAVAILABLE",
      "error": {
        "code": "DEVICE_OFFLINE",
        "message": "device is offline",
        "retryable": true
      }
    },
    {
      "adbSerial": "emulator-5554",
      "status": "READY"
    }
  ]
}
```

- Both routes require the session bearer token and accept no query parameters
  or body content.
- `GET /devices` reads the last successful in-memory result and performs no
  ADB command. Before the first successful refresh it returns an empty array.
- `POST /devices/refresh` starts or joins a refresh. A successful refresh
  replaces and returns the inventory, including an empty array when no
  devices are found.
- An ADB discovery or operation failure returns `503 ADB_UNAVAILABLE` with
  `retryable: true` and leaves the previous snapshot available to `GET`.
- `status` is exactly `READY` or `UNAVAILABLE`. `error` is absent for `READY`
  and required for `UNAVAILABLE`.
- Items are sorted lexically by `adbSerial`.

No endpoint returns an alias, display name, model, provider, physical/emulator
type, platform, screen, capability list, occupancy flag, device UUID, or device
history.

## 8. Run admission and idempotency

### `POST /runs`

The Development request contains exactly:

```json
{
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "runtimeIr": {},
  "devices": [
    {
      "adbSerial": "emulator-5554",
      "inputs": {
        "tapY": 100
      }
    }
  ]
}
```

Rules:

- `requestId` is a client-created lowercase UUID v4, persisted uniquely on the
  A5 RunGroup. One value represents one intended run.
- A repeat with an admitted `requestId` returns that RunGroup's current
  snapshot with HTTP `200` and never starts another execution. Runtime does not
  reinterpret the replacement body.
- `runtimeIr` is the exact A2 document and remains subject to its 1 MiB limit.
- `devices` contains a non-empty ordered selection with unique exact serials
  for a new request. There is no fixed count ceiling. Existing request-ID replay
  does not reinterpret replacement selection semantics after strict transport
  checks. Hardware determines practical capacity; the array envelope is unchanged.
- Each `inputs` object must satisfy the admitted A2 workflow input schema.
- The complete request is limited to 4 MiB.
- Runtime revalidates all devices and all G1 admission rules before creating
  any execution record. Run-level failure creates no partial run.

Mode, concurrency, failure policy, run name, handler versions, credential
values, artifact paths, and Production authorization are not client-selected
MVP fields. Execution capacity equals the admitted device count. Customers
control resource usage through device selection; there is no separate maximum
in YAML, environment, CLI, client settings or this request. No fixed count ceiling,
silent deduplication/filtering or automatic CPU/RAM-based reduction is permitted.
H4 transfers the complete admitted list to H3 in one owned handoff. A5 owns device failure isolation.

Admission success returns an `ExecutionSnapshot`. Semantic rejection returns
an A6 top-level error and, when validation produced actionable diagnostics, a
bounded `validation` object using the result shape in section 6.

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "run admission failed",
    "retryable": false
  },
  "validation": {
    "status": "INVALID",
    "errors": [],
    "warnings": [],
    "truncated": false
  }
}
```

## 9. Execution reads and lifecycle control

An `ExecutionSnapshot` is:

```json
{
  "cursor": "42",
  "runGroup": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RUNNING",
    "createdAt": "2026-10-22T08:00:00.000Z",
    "finishedAt": null,
    "summary": {
      "total": 1,
      "queued": 0,
      "running": 1,
      "paused": 0,
      "succeeded": 0,
      "failed": 0,
      "cancelled": 0,
      "interrupted": 0
    }
  },
  "deviceExecutions": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "adbSerial": "emulator-5554",
      "status": "RUNNING",
      "nextNodeId": "tap_01",
      "createdAt": "2026-10-22T08:00:00.000Z",
      "startedAt": "2026-10-22T08:00:00.010Z",
      "finishedAt": null,
      "terminalError": null
    }
  ]
}
```

The snapshot and `cursor` watermark are read consistently. Events committed
after the snapshot have greater cursors. Summary values are derived A5
projections, not stored counters. Device executions are ordered by creation
order and then ID.

`GET /runs/active` returns the one non-terminal snapshot or
`RESOURCE_NOT_FOUND`. `GET /runs/{runGroupId}` returns a known run regardless
of terminal state.

Attempt history is intentionally separate so snapshots remain bounded:

```http
GET /api/v1/device-executions/{deviceExecutionId}/attempts?afterSequence=0&limit=100
```

The default and maximum `limit` are 100. Results are ordered by A5 `sequence`
and contain exact A5 NodeAttempt projections:

```json
{
  "attempts": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "deviceExecutionId": "550e8400-e29b-41d4-a716-446655440002",
      "sequence": 1,
      "nodeId": "start_01",
      "attemptNumber": 1,
      "status": "SUCCEEDED",
      "startedAt": "2026-10-22T08:00:00.020Z",
      "finishedAt": "2026-10-22T08:00:00.021Z",
      "output": {},
      "error": null
    }
  ],
  "nextAfterSequence": null
}
```

`nextAfterSequence` is the last returned sequence when another page exists and
is null otherwise. Unknown query parameters are rejected.

### `POST /runs/{runGroupId}/actions`

The strict body is one of:

```json
{ "action": "PAUSE" }
```

```json
{ "action": "RESUME" }
```

```json
{ "action": "CANCEL" }
```

An accepted asynchronous action returns HTTP `202` with the current snapshot.
The later A6 event and snapshot state confirm completion. Repeating an action
whose target state is already achieved returns HTTP `200` with the current
snapshot and has no additional effect. An ineligible transition returns
`409 INVALID_EXECUTION_STATE` with exactly `error` and `snapshot` fields.

Pause, resume, and cancel apply to the complete RunGroup. The MVP exposes no
per-device lifecycle endpoint.

The [H13 implementation specification](../h_multi_device_execution/h13_lifecycle_action_api.md)
defines strict decoding, controller dependencies, complete examples and response
ordering. H13 is implemented and Linux-verified; the [shared action guide](../integration/run_actions.md)
provides the current client reference. Pending/achieved classification comes from the
live lifecycle result, not a snapshot-only status check. An accepted submission
returns 202 even if its subsequent snapshot already reflects completion or a
later action. A cancellation remains pending until H12 joins cleanup, including
when SQLite already shows CANCELLED. After that group retires, repeated CANCEL
against its saved CANCELLED record returns 200 without changing it.

## 10. SSE replay and live delivery

`GET /events` requires `Accept: text/event-stream` and the bearer header.
Native clients must use an HTTP streaming client that can set headers; the
token is never moved into the URL for browser `EventSource` convenience.

For every persisted A6 event, Runtime sends:

```text
id: 42
event: NODE_ATTEMPT_STATE_CHANGED
data: {"cursor":"42","type":"NODE_ATTEMPT_STATE_CHANGED","occurredAt":"2026-10-22T08:00:01.000Z","runGroupId":"550e8400-e29b-41d4-a716-446655440001","deviceExecutionId":"550e8400-e29b-41d4-a716-446655440002","nodeAttemptId":"550e8400-e29b-41d4-a716-446655440003","nodeId":"tap_01","payload":{"status":"RUNNING","attemptNumber":1}}

```

- SSE `id` equals the A6 `cursor`; `event` equals the A6 event type; `data` is
  the complete single-line A6 event JSON.
- `Last-Event-ID` is the only replay input. A valid value replays every retained
  event with a greater cursor, then continues live without a replay/live gap.
- Without `Last-Event-ID`, the stream starts after a connection-time watermark
  and does not replay historical events.
- An invalid cursor returns `400 INVALID_REQUEST` before streaming.
- An expired cursor returns `410 EVENT_CURSOR_EXPIRED` before streaming. The
  client gets a watermarked run snapshot and reconnects with that cursor.
- Runtime sends a comment heartbeat such as `: heartbeat` after 15 seconds of
  inactivity. Heartbeats have no ID or data and are not A6 events.
- A slow client is disconnected rather than silently losing an event. It
  reconnects using its last processed cursor.
- Ordinary request write timeouts must not terminate a healthy stream; the SSE
  implementation uses heartbeat-aware stream write deadlines instead.
- SSE responses disable intermediary buffering and compression and use
  `Cache-Control: no-store`.

The stream is Runtime-wide and unfiltered. The client filters using event
correlation fields. A7 does not add per-device cursors, log messages, command
events, or artifact events.

## 11. Artifact retrieval

`GET /artifacts/{artifactId}` returns the binary content owned by D5. For the
current node set, the only exposed artifact type is a validated PNG screenshot.

- Success uses `Content-Type: image/png`, the stored content length, and
  `Cache-Control: no-store`.
- The filename, filesystem path, storage root, checksum, and other local
  metadata are not exposed in headers or JSON.
- Range requests, artifact upload, deletion, listing, arbitrary path access,
  UI dumps, and log downloads are not part of v1.
- Missing metadata or content returns `RESOURCE_NOT_FOUND`; Runtime never
  accepts a caller-provided path.

## 12. Reduced and deferred surface

| Kept because the MVP needs it | Reduced or deferred |
|---|---|
| Minimal unauthenticated liveness | Build/host diagnostics in the handshake |
| Authenticated compatibility info | Client registration and mutable sessions |
| Backend-compatible graph diagnostics | A second diagnostics format |
| Cached inventory plus explicit refresh | Persistent devices and descriptive metadata |
| Idempotent complete-workflow start | Test-one-node, run-from-node, run name, client policy |
| Watermarked current snapshot | Unbounded attempt history inside every snapshot |
| One paged attempt-history read | Generic execution-record CRUD |
| One group action endpoint | Per-device pause/resume/cancel endpoints |
| One Runtime-wide SSE stream | WebSocket, filters, log streaming, command events |
| Screenshot download by artifact ID | Upload/list/delete/path APIs and future artifact types |
| Development source compilation and Runtime IR run request | Production package, license, lease, and authorization APIs |

A7 defines wire contracts only. B5/B6 implement authentication and the minimal
server, C5 implements validation, C9 implements compilation, E7 implements
devices, D/G/H implement execution reads and commands, D3 implements replay,
D5 implements artifacts, and I1 integrates Studio.

## 13. Checkout condition

A7 is complete when Studio can authenticate without putting credentials in a
URL, verify Runtime compatibility, validate an Editor Graph, compile source
content into complete Runtime IR without persistence or execution side effects,
refresh and select ADB endpoints, safely retry one start request, read and
control its run, replay typed events from a Runtime-wide cursor, recover from
expiry using a consistent snapshot, and retrieve a screenshot without access
to a local path.

A8 provides the [portable Local API and SSE cases](./contract_fixtures_v1.md)
used by later Runtime and client contract tests.

## H4 selection-availability diagnostics

A 422 readiness rejection preserves the existing top-level first availability
code and adds optional `unavailableDevices`, ordered by submitted index. Each
entry is `{index, adbSerial, reason: {code, message, retryable}}`; index is
zero-based, reason is the checked E6 safe diagnostic, and retryable is true.
Return every unavailable selected entry, no accepted subset or arbitrary inventory.
Other responses omit this field; IR/input errors retain the existing validation
envelope. See [H4](../h_multi_device_execution/h4_public_multi_device_start.md)
for the full request/response and replay/whole-selection contract.
