# H4 — Public multi-device start

## Overview

H4 exposes H3's complete-group coordinator through the existing Development
`POST /api/v1/runs` API. A publisher selects one or more READY devices and
submits one compiled Runtime IR with a separate input object for each serial.
Runtime validates the complete selection, commits one RunGroup with all its
device executions, transfers ownership once, and returns a snapshot. Workers
continue independently of the HTTP connection.

Selecting two devices creates two concurrent executions; selecting one still
works. There is no fixed device-count ceiling or separate concurrency setting.
Transport byte limits, node/input contracts and readiness deadlines still apply.
All selected devices are admitted together or none are admitted. Runtime never
silently removes unavailable entries or deduplicates different input objects.

```mermaid
sequenceDiagram
    participant C as Studio native client
    participant API as POST /runs
    participant A as G12 admission and E6 readiness
    participant DB as D2/D4 SQLite
    participant G as H3 group owner
    C->>API: requestId + runtimeIr + ordered devices/input pairs
    API->>A: Existing complete-group Admit
    alt requestId already admitted
        A->>DB: Read saved group and complete snapshot
        API-->>C: 200 saved snapshot; no workers created
    else New request
        A->>A: Validate every input, serial and fresh readiness
        alt Invalid or unavailable
            API-->>C: 400/413/422; no partial run
        else Whole selection valid
            A->>DB: Atomically admit group, N devices and events
            API->>G: Transfer all N owned requests once
            G->>G: Register independent device workers
            API-->>C: 201 committed snapshot
            G->>DB: Existing per-device checkpoints and group aggregation
        end
    end
    C->>API: GET /runs/groupId
    API-->>C: Current complete snapshot; no completion wait
    Note over G: Owner stop fences dispatch and joins all workers through G11
```

**Status — Implemented and Linux-verified on 2026-11-22.**
H1–H3 and no-ceiling admission/storage foundations already exist. H4 removes
the temporary public single-device policy and updates consumer documentation.
It introduces no new execution engine, endpoint, permission or schema migration.
Production encrypted-package admission/authorization remains its separate work;
Launcher must not send customer package plaintext to this Development endpoint.

## 1. Scope and prerequisites

| Owner | Responsibility |
| --- | --- |
| A7 / runs transport | Existing authenticated loopback JSON API, strict DTO parsing, byte limits and HTTP projection. |
| G1/E6/G12 | Runtime IR and all-device input validation, ordered fresh readiness and whole-selection admission. |
| D2/D3/D4/D6 | Atomic records/events, replay, complete snapshots, authoritative result aggregation and crash interruption. |
| H1–H3 | One group owner, copied per-device work, membership-derived reservations, isolated outcomes and all-worker cleanup. |
| H4 | Replace the single-device admission/handoff policy, expose actionable selection rejections and publish verified client examples. |
| Later H steps | Disconnect policy, waits, pause/resume, live cancellation and lifecycle-action APIs. |

Keep one active non-terminal RunGroup. Multiple devices inside that group do
not permit multiple simultaneous groups. Preserve owner-managed process lifetime,
current Bearer authentication, per-request readiness checks and existing logging.

## 2. Request contract

`POST {apiBaseUrl}/runs` still requires JSON, the current session Bearer token,
no query, no content encoding and exactly these root fields:

| Field | Contract |
| --- | --- |
| `requestId` | Canonical lowercase UUID v4 generated once per intended run; retain across network retries. |
| `runtimeIr` | Exact object returned by compilation; at most 1 MiB. Do not submit the compilation envelope, graph or stringified JSON. |
| `devices` | Ordered array of `{adbSerial, inputs}` objects. A fresh request requires at least one entry, without a product count ceiling. |
| `devices[i].adbSerial` | Exact selected serial including its TCP port if present; valid, unique and freshly READY at admission. |
| `devices[i].inputs` | An object satisfying that IR's input schema. Each entry can have different values; `{}` for an empty schema. |

The entire UTF-8 body remains bounded by 4 MiB, including envelope and whitespace.
Unknown fields, duplicate JSON keys, bad types/nulls, invalid UTF-8 and trailing
JSON retain the existing strict rejection behavior. Do not add a maximum count,
concurrency field, variable seed map, target-mode override or Production field.

The selected list controls resource usage. Runtime binds each serial/input pair
to its own device UUID in request order; discovery order must not reorder it.
Unselected inventory devices never create workers. A change to the next run's
selection requires no Runtime restart.

## 3. Admission, replay and error behavior

Keep G12's request-ID lookup before replacement executable/device semantics.
Strict authentication, envelope parsing and byte bounds still precede service
lookup. An existing ID can resolve its saved group even when replacement IR or
selection would fail new-run validation. Clients should resend the retained
original body rather than exploit placeholders or empty replacement arrays.

| Result | HTTP / effect |
| --- | --- |
| New valid complete selection | 201 direct snapshot after commit and owned group handoff; one admission and one handoff. |
| Existing requestId | 200 current saved snapshot; no readiness scan, new scope, worker or event from the replay itself. |
| Empty fresh selection, malformed/duplicate serial or invalid envelope | 400 INVALID_REQUEST, with no partial admission. |
| Byte budget exceeded | 413 PAYLOAD_TOO_LARGE; do not misclassify it as too many devices. |
| IR/input incompatibility | Existing 422 validation envelope with bounded `devices[i].inputs` paths where applicable. No workers or partial records. |
| Selected endpoints unavailable | 422 existing first availability code plus ordered `unavailableDevices` diagnostics for every unavailable selected entry. No selected device starts. |
| Another group active | 409 RUNTIME_BUSY under existing replay/busy ordering. |
| ADB observation failed | 503 ADB_UNAVAILABLE; distinct from successfully observed unavailability. |
| Admission outcome unknown or handoff contract failed | Existing persistence/internal failure and Runtime failure fence. Never redispatch to resolve uncertainty. |

Validation failure does not become partial success. Preserve the existing
ordering of G1, E6 and D2; do not add database probes or a second scheduler in
the HTTP layer. Runtime may become busy while preflight runs; D2 remains the
atomic authority for competing admissions.

### Availability diagnostics

Project the existing E6 `DeviceSelectionIssue` values into a typed API DTO;
do not expose Go field names or raw ADB output. Add an optional field only for
availability rejection, preserving existing error/validation envelopes:

```json
{
  "error": {
    "code": "DEVICE_OFFLINE",
    "message": "run admission failed",
    "retryable": true
  },
  "unavailableDevices": [
    {
      "index": 1,
      "adbSerial": "emulator-5556",
      "reason": {
        "code": "DEVICE_OFFLINE",
        "message": "device is offline",
        "retryable": true
      }
    }
  ]
}
```

Indices are zero-based positions in the submitted array, ordered ascending.
The top-level code remains the first unavailable member's existing error code.
Return only selected unavailable entries, never an accepted subset or arbitrary
inventory. Reuse E6's checked, safe messages. Omit `unavailableDevices` on other
responses and omit `validation` when no validation result exists. Body size
scales with the bounded submitted selection; do not introduce a hidden count cap.

## 4. Start-service and group ownership changes

1. Replace `SingleDeviceAdmitter.AdmitSingleDevice` with an `Admitter.Admit`
   interface implemented by the existing G12 service. Remove the obsolete
   single-device service entry point, private maximum argument and constant.
2. Keep the outer G11 start lease across admission and the whole ownership
   transfer. After confirmed CREATED, request cancellation cannot discard work.
3. Validate the complete prepared work contract before handoff: non-empty list,
   runner/group identity, every member's matching group, valid unique UUID/serial,
   non-nil input map and no preinstalled Control/store/variable view authority.
   A malformed later member is an internal failure, not a partial handoff.
4. Dispatch the entire ordered list exactly once through H3's group owner.
   Do not index only member zero, loop HTTP starts or copy one input to siblings.
5. Remove `DispatchGroup`'s one-device policy while preserving H3's complete
   validation/registration, failure fences, shared cleanup and prior-group join.
   Its private core can become the ordinary public entry point without another
   scheduler layer or exported bypass.
6. Release the start lease only after ownership transfer, then read the snapshot
   using existing G13 projection. A lost response or snapshot error cannot
   cancel or resubmit the already-owned workers.
7. Update Wire bindings and regenerate `wire_gen.go`; never edit generated code
   manually. No constructor configuration or new dependency is needed.

Keep H3 registration and stop linearization. Stop after admission but before
registration owns all queued rows through the start lease; G11 intentionally
settles them or D6 interrupts after a crash. Replay, rejected input, busy and
rolled-back admission must not enter group dispatch.

## 5. Response and persistence behavior

POST returns promptly after admission/handoff; it does not stay open for workflow
completion. Its direct `ExecutionSnapshot` has the existing cursor, group summary
and ordered deviceExecutions array. A fast workflow can already be terminal in
the 201 snapshot. HTTP 201 means newly admitted, never proof of success.

Both existing GET routes return current committed progress immediately:
`GET /runs/active` and `GET /runs/{runGroupId}`. They are polling JSON APIs, not
SSE, and include every stored member. Group state/counters come from D4/G13.
Preserve device failure isolation, Flow/input/output isolation and explicitly
shared Global variables. One successful member cannot close sibling RAM.

H4 adds no tables, migrations, duplicated counters or per-node database work.
Admission creates one group and N devices atomically, with existing bounded
write batches inside one transaction. Retries/GETs create no extra work; active
workers may independently commit while a snapshot/replay is being served.

## 6. Client integration and examples

Update the shared native guide and Studio guide together. Studio captures the
complete selected serial/input array alongside the graph/schema revision before
compilation and submission. Validate that every selected serial is READY, keep
one pending requestId/body across transport recovery, and never silently filter.
When an availability rejection arrives, highlight its indexed entries, refresh,
and let the user correct the selection. Ordinary HTTP errors do not trigger
an automatic Runtime restart; use the existing readiness/replacement rules.

The Launcher guide must distinguish this Development route from pending
Production encrypted-package authorization/start. Multi-device Development
acceptance does not authorize sending plaintext customer workflows.

Deliver `examples/h4/compile-workflow.json`, `start-run.json` and a short README:
Start -> input-driven Delay -> End, two serial/input pairs, no Android UI side
effects. The start file contains complete validated Runtime IR and can be pasted
into Postman. Publishers may compile the graph and replace the IR directly.
Use a fresh requestId for a new run. Document readiness/token prerequisites,
201/200 semantics, polling, invalid later member, duplicate/empty selection,
per-device input failure and expected SQLite counts as a manual test aid.

For the supplied three-node workflow, N successful devices produce N*3 attempts
and 11*N+2 committed events: admission N+1, ten per-device events and one terminal
group event. Thus two devices finish with one group, two devices, six attempts
and 24 events. Concurrent event ordering can interleave; do not require a fixed
snapshot cursor or handler completion order. Counts are scoped to that group.

A normal Waydroid instance provides only one independent target. Keeping one
entry in the example remains valid; do not advertise multiple fictional serials
as real hardware acceptance. Controlled ADB tests verify the transport/engine;
independent real-device/native Windows acceptance remains separately recorded.

## 7. Implementation locations and review scope

| Runtime path | Change |
| --- | --- |
| `internal/services/runadmission/service.go` | Remove single-device wrapper/maximum; keep replay-first existing Admit. |
| `internal/services/runadmission/start.go` | Use complete-group Admitter and verify every prepared member before one handoff. |
| `internal/services/executionlifecycle/group.go` | Remove public cardinality guard and reuse H3 coordinator. |
| `pkg/constants/api.go` | Remove `MaxStartDevices`, without a replacement count limit. |
| `cmd/agent/wire.go`, generated `wire_gen.go` | Bind the complete-group admission interface and regenerate. |
| `internal/api/runs/dto.go`, `start.go` | Typed optional availability diagnostics from existing E6 results. |
| Existing start/admission/lifecycle/API tests | Replace temporary single-device rejection expectations; keep auth, byte, replay, ownership, cancellation and snapshot regressions. |
| Focused H4 API/process tests and examples | Real HTTP/SQLite/agent composition with controlled endpoints and runnable client payloads. |

The transport already uses an array; do not replace strict decoding or add a
new execution endpoint. Keep production logic focused; no H5 disconnect watcher,
H6-H13 wait/control feature, Backend/client implementation or Production crypto.

## 8. Approved test scope and verification

The user approved focused H4 API/process tests before implementation. Reuse
existing fixtures, the concrete H3 coordinator and normal router/agent wiring.

| Scenario | Required evidence |
| --- | --- |
| Multi-device new start | 201 before blocked work finishes; all selected workers enter with their own inputs; snapshot order matches request, not discovery. |
| Larger selection | An above-32 fixture passes without a replacement ceiling; every admitted member completes once. |
| Active/terminal replay and concurrent duplicate POSTs | One group/handoff per requestId; exactly one creator; no new executions, identities or stored events from replay. |
| Whole-selection rejection | Empty/duplicate/malformed serial, bad later input and unavailable later endpoint cause zero partial records or handlers. Availability response identifies all failing indices. |
| Prepared-work contract corruption | A later mismatched/prebound/duplicate member fences the start rather than transferring a valid prefix. |
| Stop/disconnect around admission and handoff | Confirmed ownership survives request cancellation; intentional stop settles every pending/active member with no orphan or repeated action. |
| Observability/failure isolation | GET snapshots include all devices; a durable workflow failure leaves siblings running and D4 derives the final result. |
| Actual-agent process acceptance | Compiled example through actual router, authentication, ADB readiness, POST/GET/replay and SQLite. Use controlled ADB, private owner pipes and isolated state; no production test hook. |

Run focused and full/race tests, formatting, `make wire`, build, vet, diff/link
checks, managed-process acceptance and Windows amd64 cross-build. Report native
Windows and independent real Android execution separately. The missing G22
Python harness remains a known full-suite failure unless restored; do not remove
or silently pass that test as part of H4.

### 8.1 Current execution evidence

Verification on 2026-11-22 used the actual build and temporary SQLite stores;
no customer database or Android app was modified.

| Check | Result |
| --- | --- |
| Focused admission/lifecycle/API tests | Pass; includes complete 2/33-device handoff, serial/input pairing, rejection with zero writes, indexed availability diagnostics, active/terminal replay and prepared-work/stop races. Existing H3 failure-isolation tests reuse the public coordinator. |
| `go test ./cmd/agent -run '^TestH4MultiDeviceAgentAcceptance$' -v -count=1` | Pass. The Linux agent compiles the readable example and executes 2 and 33 controlled READY endpoints through authenticated HTTP. |
| Actual-agent SQLite observations | First 2 devices: 1 group, 2 devices, 6 attempts, 24 events. After a second 33-device group: totals 2 groups, 35 devices, 105 attempts, 389 events. Lost response resolves by replay; rejected selections add no records. |
| Owner stop/restart | Two active delays settle CANCELLED before clean exit; restart GET returns the saved result without dispatch or new records. |
| `go test ./... -count=1` | All packages except cmd/agent pass; the existing TestG22CombinedEngineAcceptance fails because cmd/agent/testdata/g22_acceptance.py is absent. H4 acceptance passes in that same package. No test was silently removed or rewritten to hide this failure. |
| `go test -race ./... -skip '^TestG22CombinedEngineAcceptance$' -count=1` | Pass for the available suite. The first run found the shared test helper's 2-second settlement deadline too short for the 33-device race fixture; the test-only deadline is now 10 seconds. Production timeouts are unchanged. |
| Existing managed-parent acceptance | Pass: readiness/authentication, competing launches, shutdown, owner EOF, external kill and shared replacement after probe timeout. |
| Formatting/wiring/build/static checks | `make fmt`, `make wire`, `make build`, and `go vet ./...` pass. Wire regeneration requires no generated diff. |
| Windows amd64 | Agent and affected API/admission/lifecycle tests cross-compile. No native Windows execution claimed. |

`cmd/agent/testdata/h4_acceptance.py` is a controlled ADB/HTTP/process test, not
real Android acceptance and not a replacement for the missing G22 harness.
The [manual example](../../../../a2matex-runtime/examples/h4/README.md) is ready
for selected real devices. Two independent Android targets and native Windows
remain H15 acceptance work. H4 functionality is delivered; the missing G22
harness remains a separate open regression gate.

## 9. Completion and deferred work

H4 is complete when the public route admits a non-empty ordered selection,
transfers every member to H3 once, preserves whole-selection rejection and
replay/ownership semantics, returns complete snapshots, and ships accurate
client examples and verification evidence. Existing one-device clients remain
valid because their one-item array satisfies the same contract.

H5 owns explicit disconnect handling. H6-H8 own waits; H9-H13 own pause/resume
and live cancellation; H14-H15 broaden shutdown/native multi-device acceptance.
Production package authorization remains its own phase. No additional maximum,
settings API, persisted queue, automatic restart/resume or SSE is introduced.
