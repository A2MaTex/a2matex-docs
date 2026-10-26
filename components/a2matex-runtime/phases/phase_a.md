# Phase A — Contract Freeze

Phase A defines the shared contracts used by Runtime, Studio, and eventually
Launcher. It does not implement workflow execution, devices, persistence, or
node handlers.

## Progress

| ID | Module | Status |
|---|---|---|
| A1 | Contract decision register | **Accepted** |
| A2 | Runtime IR v1 contract | **Accepted** |
| A3 | Session device selection rules | **Accepted** |
| A4 | Typed device command contract | **Accepted** |
| A5 | Execution schemas | **Accepted** |
| A6 | Event and error schemas | **Accepted** |
| A7 | Local API schema | **Accepted** |
| A8 | Contract fixtures | **Accepted** |

## A1 — Contract Decision Register

**Status:** Accepted

| Area | Accepted MVP decision |
|---|---|
| Versioning | Runtime IR uses integer `formatVersion: 1`. Runtime application version and `minimumRuntimeVersion` use SemVer. Node-handler versioning is deferred. |
| Local API | HTTP/1.1 with JSON on configurable loopback address `127.0.0.1:8765`, versioned under `/api/v1`. |
| Events | Server-Sent Events with replay. Each persisted event has a Runtime-wide increasing cursor, exposed as an opaque decimal string. |
| Time | JSON timestamps use UTC RFC 3339 with fixed millisecond precision. Durations use numeric milliseconds measured by Runtime. |
| ADB | Resolution order is configured path, Runtime-bundled ADB, then development `PATH`. Runtime selects and validates one executable per process. |
| Android devices | A Runtime session discovers and selects current ADB endpoints; emulator-provider integration is deferred. |
| Artifacts | Binary artifacts use a Runtime-controlled directory under the OS application-data location, with a configurable override. Workflow data cannot choose filesystem paths. |
| Runtime IDs | Persisted Runtime-created execution entities use opaque lowercase UUID v4 values. Session devices are not Runtime-created entities. |
| Supplied IDs | Backend IDs are preserved. Node and edge IDs are bounded opaque strings supplied by Runtime IR and unique within their document scope. |
| Execution admission | At most one non-terminal run group exists; a second start request is rejected while it is active. |

### Required consequences

- `formatVersion`, Runtime application version, and `minimumRuntimeVersion` are
  independent compatibility mechanisms.
- Registry lookup remains `runtimeHandler -> handler` for the MVP.
- Runtime rejects unsupported Runtime IR versions instead of guessing or
  silently converting them.
- The event cursor controls ordering and replay; timestamps are not cursors.
- Authoritative execution state and all resulting events are persisted before
  SSE subscribers are notified.
- API credentials are never placed in query parameters.
- ADB commands target an explicit serial and are executed without a shell.
- Current ADB serials identify selectable devices only within the Runtime
  session; they are revalidated before execution and are not persisted as a
  Device entity.
- The persistence engine and database file location remain undecided until
  Phase D.

### Deferred contract details

- A2 defines Runtime IR fields, limits, and node/edge identifier constraints.
- A3 defines session discovery and selection rules only.
- A5 stores the accepted ADB serial on each `DeviceExecution`.
- A6 defines event envelopes, cursor expiry, retention, and error contracts.
- A7 defines Editor Graph validation, device discovery/start payloads, endpoint
  paths, authentication, SSE framing, and reconnection.
- A8 provides shared Editor Graph, Runtime IR, event/error, Local API, and SSE
  fixtures so consumers can verify parity without sharing implementation code.

Changes to an accepted A1 decision require an explicit replacement decision
and an impact review across Runtime, Studio, Launcher, and Backend.

## A2 — Runtime IR v1 Contract

**Status:** Accepted

Runtime IR is the self-contained executable document compiled by Studio and
stored in `ProjectDraft.runtimeIr`. It excludes editor-only state and does not
use the full JSON Schema standard or a generic schema interpreter.

References:

- [Runtime IR v1 contract](../contracts/runtime_ir_v1.md)
- [Canonical Runtime IR v1 example](../contracts/runtime_ir_v1.example.json)

| Area | Accepted MVP decision |
|---|---|
| Envelope | `formatVersion`, minimum Runtime version, source graph metadata, entry node, input schema, nodes, edges, and derived capabilities |
| Reduced schemas | Flat object properties using `string`, `integer`, and `boolean` with only approved constraints |
| Input sources | `LITERAL`, `WORKFLOW_INPUT`, and `NODE_OUTPUT`; no expressions, interpolation, or implicit conversion |
| Graph | One Start, reachable End, acyclic, sequential per device, and only `success`/`error` transitions |
| Policies | Explicit timeout, retry, and error policy on every compiled node |
| Outputs | One Backend-owned payload schema per node; A5 later wraps it in a unified execution-result envelope |
| Limits | 1 MiB, 2–500 nodes, 1–1,000 edges, bounded fields, identifiers, retries, and timeouts |
| Validation | Strict structural checks followed by catalog, binding, graph, capability, and compatibility checks |

The contract was checked against all 11 current Backend node definitions.
Shared valid and invalid cross-language fixtures remain A8 scope.

Runtime validation has two separate reliability boundaries:

- The local Editor Graph validator gives Studio deterministic authoring
  diagnostics before compilation. It mirrors the approved Backend rules but
  validates node contracts against the Runtime's supported-node registry.
- The Runtime IR admission validator validates the exact executable document,
  workflow inputs, handler availability, and compatibility again for every run.

A previous Studio or Backend result, including `validationSummary`, is never
trusted as execution authorization. Admission failure creates no active run and
dispatches no device command.

## A3 — Session Device Selection Rules

**Status:** Accepted

Runtime discovers current ADB endpoints for selection within one Runtime
session. It does not create or persist a Device entity, stable device UUID,
device name, provider record, or device history.

| Area | Accepted MVP decision |
|---|---|
| Discovery | Runtime uses `adb devices -l` through its selected ADB executable and keeps only an in-memory current inventory |
| Selection | Clients select one or more current opaque ADB serials; duplicate serials are rejected |
| Readiness | Runtime probes every selected serial again immediately before accepting a run |
| Routing | Every Android command is bound to the accepted explicit serial; Runtime never guesses a replacement endpoint |
| Disconnect | Loss of a selected endpoint affects only its A5 `DeviceExecution`; the policy and error are defined by A5/A6 |
| Persistence | A5 copies `adbSerial` into `DeviceExecution`; no Device table, registry history, provider correlation, or device UUID exists |
| API | A7 owns discovery, refresh, selection, and validation payloads |
| Errors | A6 owns stable ADB, readiness, disconnect, and `RUNTIME_BUSY` codes |

Device names, aliases, model, physical/emulator type, provider metadata,
platform data, screen data, capabilities, availability history, occupancy, and
device timestamps are outside the MVP contract.

## A4 — Typed Device Command Contract

**Status:** Accepted

A registered Android handler creates one internal typed command from already
validated node inputs. The dispatcher binds it to the current
`DeviceExecution.adbSerial`; workflow data cannot choose the executable, raw
arguments, target serial, shell syntax, timeout, retry policy, or artifact path.

Reference:

- [Typed Device Command v1 contract](../contracts/typed_device_command_v1.md)

| Area | Accepted MVP decision |
|---|---|
| Variants | `LAUNCH_APP`, `TAP`, `SWIPE`, `INPUT_TEXT`, `PRESS_BACK`, `PRESS_HOME`, and `SCREENSHOT` |
| Key events | Back and Home are separate zero-field commands; A4 exposes no generic key-code field |
| Targeting | The command contains no serial; the dispatcher uses A5 `DeviceExecution.adbSerial` |
| Execution control | Timeout, cancellation, retry, and error policy remain outside the command |
| Process safety | The adapter invokes the selected ADB executable directly with separate fixed arguments and no general shell |
| Results | Successful non-screenshot commands have no payload; Screenshot returns only `artifactId` after storage |
| Diagnostics | Exit code, bounded stdout/stderr, and duration belong to E1/B4 process diagnostics, not A5 state, A6 events, or workflow output |

All command field constraints follow the approved Backend node contracts. A4
defines the boundary only; Phase F implements ADB arguments and real execution,
while A6 owns errors and A5 owns persisted results.

## A5 — Execution State v1 Contract

**Status:** Accepted

Runtime persists only three execution records: RunGroup, DeviceExecution, and
NodeAttempt. A checkpoint is an atomic write operation, not another entity.

Reference:

- [Execution State v1 contract](../contracts/execution_state_v1.md)

| Area | Accepted MVP decision |
|---|---|
| Run group | One active group at a time; `RUNNING`, `PAUSED`, and four terminal states |
| Device execution | One isolated record per accepted serial; the immutable `adbSerial` is copied from A3 |
| Node attempt | One record per real attempt with per-device sequence, attempt number, validated output, and optional A6 error |
| Checkpoint | Atomically finishes the attempt, updates device/group state, and persists all resulting A6 events before broadcast |
| Retry | A new attempt record; no `RETRYING` state |
| Results | Node output is persisted; resolved input, mutable variables, and process diagnostics are not A5 fields |
| Summary | Status counts and final summaries are derived from DeviceExecution and NodeAttempt records |
| Recovery | Every non-terminal group/device and running attempt becomes `INTERRUPTED`; no automatic replay |

Specialized license/offline pause states, persistent device snapshots,
configurable group failure policy, stored counters, and a separate checkpoint
entity are outside the MVP.

## A6 — Event and Error v1 Contract

**Status:** Accepted

A6 defines four durable typed execution events and one reduced error envelope.
A5 remains authoritative; events update clients from a known Runtime-wide
cursor, while an A7 watermarked current-state snapshot repairs expired history.

Reference:

- [Event and Error v1 contract](../contracts/event_error_v1.md)

| Area | Accepted MVP decision |
|---|---|
| Cursor | One Runtime-wide increasing opaque decimal string; no per-device event cursor |
| Events | RunGroup state, DeviceExecution state, NodeAttempt state, and retry scheduling only |
| Correlation | Run group always; device and node attempt only when applicable |
| Payload | Strict type-specific object capped with the complete event at 16 KiB |
| Error | `code`, bounded safe `message`, and occurrence-specific `retryable` only |
| Replay | Persist with A5 state before broadcast; replay events after the supplied cursor |
| Expiry | `EVENT_CURSOR_EXPIRED`, followed by an A7 snapshot with a cursor watermark |
| Logs | B4 owns operational logs; raw process diagnostics and generic log payloads are not events |

Validation diagnostics remain C1/A7 scope. Publisher-authored `utility.log`
persistence requires a separate sensitive-data decision before F8.

## A7 — Local API v1 Contract

**Status:** Accepted

A7 exposes the smallest authenticated Development API that lets Studio validate
a graph, discover devices, run a complete draft, observe and control it, and
retrieve screenshots. It projects A2–A6 rather than redefining their models.

Reference:

- [Local API v1 contract](../contracts/local_api_v1.md)

- All non-health routes require a bearer token delivered outside HTTP through
  a protected per-user connection file.
- Start is idempotent through a client UUID persisted on A5 RunGroup.
- State recovery uses a watermarked snapshot, paged attempt history, and one
  Runtime-wide SSE stream with `Last-Event-ID`.
- A7 adds only the four transport errors needed for invalid, unauthorized,
  missing, and oversized requests.

Production APIs, WebSocket, test-one-node, run-from-node, per-device controls,
log streaming, API device metadata, and generic execution CRUD are deferred.

## A8 — Runtime Contract Fixtures

**Status:** Accepted

A8 contains 19 versioned, hashed test vectors for the shared Editor Graph,
Runtime IR, A6, and A7 boundaries. It is representative rather than an
exhaustive replacement for phase-owned unit and integration tests.

References:

- [Runtime Contract Fixtures](../contracts/contract_fixtures_v1.md)
- [Current fixture manifest](../contracts/fixtures/v2/manifest.json)

The canonical files remain in the plan repository and are vendored unchanged
into consumer test suites. Cross-language agreement is not yet proven; C6, G1,
I1, and J must execute the fixtures in TypeScript and Go.

## Phase A gate

Phase A is complete. Before implementation continues, the existing Runtime
scaffold should be reconciled against the accepted A1–A8 contracts and the next
small Phase B checkout selected.
