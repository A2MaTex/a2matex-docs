# H13 — Group lifecycle-action API

## Overview

H13 exposes the existing internal Pause/Resume behavior and H12's implemented Cancel
through one authenticated HTTP endpoint. Studio and Launcher submit a group
action, receive a current execution snapshot immediately, then observe completion
through the existing by-ID snapshot API. The request does not wait for an Android
command, pause barrier or cancellation cleanup to finish.

For a running Start → Delay → Tap → End workflow, PAUSE can return `202` while
Delay finishes; polling later shows PAUSED with Tap next. RESUME continues the
same workers. CANCEL interrupts unfinished work and eventually releases the group
while Runtime stays alive. It does not shut down the owner's Runtime process.

```mermaid
sequenceDiagram
    participant C as Studio or Launcher native client
    participant A as H13 authenticated action handler
    participant L as H9-H12 group lifecycle
    participant R as Existing snapshot reader
    C->>A: POST /runs/{id}/actions + bearer + action
    A->>A: Strict path/body validation
    A->>L: Submit exactly one group action
    Note over L: H9-H12 internal controls implemented; public route pending
    L-->>A: ACCEPTED / ALREADY_TARGET / classified error
    A->>R: Read exact group's consistent snapshot
    R-->>A: Snapshot and committed cursor
    A-->>C: 202 accepted / 200 already achieved / 409 conflict
    L->>L: Owned async work, checkpoints and cleanup
    C->>R: GET /runs/{id} until paused, runnable or terminal
    Note over C,R: SSE is Phase I; owner shutdown remains B3/G11
```

**Status — H13 implemented and Linux-verified on 2026-11-23.**
Section 12 records current evidence and platform limits. Focused H13 test scope
was approved before adding tests; further additions retain that approval rule.
H12 is now implemented and Linux-verified; section 2 retains the frozen dependency
contract established before its source existed. HTTP decoding/mapping tests can
use a narrow controller double. The complete endpoint must not be advertised or
accepted as finished until real H12 cancellation passes integration checks.
No placeholder Cancel implementation or partially supported action endpoint is
authorized by this specification.

## 1. Scope, current code and dependencies

| Current area, relative to `a2matex-runtime` | H13 responsibility |
| --- | --- |
| `internal/api/router.go` | Register one POST inside the existing `/api/v1` group. |
| `internal/api/runs/handler.go`, `dto.go` | Add a narrow controller dependency, action request DTO and conflict response DTO; reuse SnapshotReader and SnapshotResponse. |
| `internal/api/runs/decode.go` | Reuse strict JSON/media/duplicate-key helpers; add an action-specific bounded reader. |
| `internal/services/executionlifecycle/control.go` | Consume implemented Pause/Resume; do not implement another pause barrier. |
| Implemented [H12 cancellation](h12_run_cancellation.md) | Consume the agreed Cancel method and its live ownership/cleanup guarantees. |
| `internal/api/response` and middleware | Preserve authentication, no-store, contextual errors, recovery and one request-log boundary. |
| `cmd/agent/wire.go` | Bind the new controller interface to the existing lifecycle singleton and regenerate Wire. |

The business sources remain [A7 transport](../contracts/local_api_v1.md#9-execution-reads-and-lifecycle-control),
[A5 states](../contracts/execution_state_v1.md),
[A6 errors/events](../contracts/event_error_v1.md),
[H9–H11](h9_h11_pause_resume.md), [H12](h12_run_cancellation.md),
and [G13 snapshots](../g_workflow_engine/g13_run_snapshot_apis.md).

No action table, operation resource, migration, new execution status, event broker,
ADB discovery, workflow validation, start lease or worker dispatch belongs here.
Actions target all admitted members; there is no per-device action API.

## 2. Frozen dependency contract

Define the controller interface in its consumer package `internal/api/runs`:

```go
type Controller interface {
    Pause(context.Context, string) (constants.GroupControlDisposition, error)
    Resume(context.Context, string) (constants.GroupControlDisposition, error)
    Cancel(context.Context, string) (constants.GroupControlDisposition, error)
}
```

These are the existing Pause/Resume signatures and the signature specified by
H12. Do not introduce an HTTP-aware lifecycle method or require H12 to build
snapshots. The handler uses its existing `SnapshotReader.SnapshotByID` afterward.

| Dependency guarantee | Provider | Consumer rule |
| --- | --- | --- |
| ACCEPTED means one owned async action, including a duplicate still pending. | H9–H12 | Return 202 after reading a snapshot; do not wait or submit again. |
| ALREADY_TARGET means the target was achieved at the lifecycle ordering point with no conflicting pending action. | H9–H12 | Return 200 with a freshly read snapshot. |
| Pending opposite Pause/Resume is an invalid transition. | H9–H11 | Return 409; do not infer eligibility from an older SQLite status. |
| Cancel supersedes pending Pause/Resume and blocks later controls. | H12 | Submit CANCEL once; do not orchestrate Pause/Resume first. |
| Duplicate Cancel remains ACCEPTED while settlement/cleanup is pending, even if SQLite already says CANCELLED or group context is cancelled. | H12 | Consult controller before interpreting a terminal snapshot as an achieved duplicate. |
| Retired groups need no live cache; missing/different/ineligible owner returns `ErrExecutionControlState`. | H9–H12 | Read exact saved target to distinguish 404, achieved historical cancellation and 409. |
| Stopping and real storage/invariant failures are classified separately from invalid state. | H9–H12 | Never convert those failures into 200/409 using a saved snapshot. |
| Submission context cancellation after acceptance does not undo owned work. | H9–H12 | A failed HTTP response is not evidence that the action was rejected. |

The H12 implementation handoff is complete when these semantics hold for its
real Cancel method, including cleanup and pending-control races. Exact private
field/helper names are free to follow the final implementation. No extra H12
method, status-probe API or durable intent store is needed for H13.

`COMMITTED` and `STALE` are checkpoint-layer dispositions, not expected results
of these public lifecycle methods. A nil-error unknown disposition is a trusted
dependency-contract failure: respond 500, never assume success.

## 3. Route and strict request contract

```http
POST /api/v1/runs/550e8400-e29b-41d4-a716-446655440001/actions
Authorization: Bearer <current Runtime session token>
Content-Type: application/json

{"action":"PAUSE"}
```

Valid bodies are exactly `{"action":"PAUSE"}`, `{"action":"RESUME"}` and
`{"action":"CANCEL"}`. Whitespace around JSON syntax is allowed; values are
case-sensitive and are not trimmed or normalized.

| Input | Required handling |
| --- | --- |
| Path ID | Use `execution.ValidID`: canonical lowercase UUID v4, without another regex, trimming or second URL decode. |
| Query | Reject any nonempty RawQuery; no filters, force option, timeout or request ID. |
| Media type | Require `application/json`; optionally accept only `charset=utf-8`, case-insensitive charset value, matching current POST handling. |
| Content-Encoding | Reject nonempty encodings; do not transparently decompress. |
| Body size | At most 1,024 bytes, including whitespace. Add a named action-body constant to `pkg/constants/api.go`; enforce for known and unknown ContentLength with MaxBytesReader. |
| JSON | One valid UTF-8 JSON object, exactly one required string field `action`; reject duplicates, escaped-equivalent duplicate keys, unknown fields, trailing JSON, arrays and null. |
| Action | One of the three exact uppercase strings. Reject missing/null/numeric/empty/lowercase/padded values. |
| Unsupported method or extra path segment | Preserve existing router/no-route behavior; add no redirects or general method-policy refactor. |

Authentication runs before route validation or controller/snapshot access.
Invalid transport input invokes neither controller nor snapshot reader. Invalid
body/media/path/query is 400; oversized body is 413. For a malformed media type
and oversized body together, follow current POST validation order: media/query
validation, declared-size check, bounded read, JSON validation.

Do not reuse bodyless GET `validRequest` for this route. Reuse existing
`uniqueKeys`, `objectFields` and `stringField` helpers where they fit; no new
generic decoder framework or dependency is required.

## 4. DTO, injection and handler algorithm

Add discoverable DTOs in `internal/api/runs/dto.go`:

```go
type ActionRequest struct {
    Action string `json:"action"`
}

type ActionConflictResponse struct {
    Error    apiresponse.Error `json:"error"`
    Snapshot SnapshotResponse  `json:"snapshot"`
}
```

Use named shared action constants for branching and validation. A small named
string type is also acceptable if it follows existing conventions. Success
responses are directly `SnapshotResponse`, not `{snapshot: ...}`. Only the 409
response has the `error` plus `snapshot` wrapper.

Extend constructor injection to `NewHandler(reader, starter, controller)` and
adapt existing fixtures explicitly. Bind `runs.Controller` to the same
`*executionlifecycle.Service` already used for starts and shutdown. Missing
trusted dependencies are 500, not empty results, 404 or a disabled Cancel success.
Regenerate `wire_gen.go`; do not edit it manually.

For one valid request:

1. Select the controller method from the validated action. Call it **exactly
   once** with the request context and exact target ID. Do not read status first
   as an authorization decision, acquire a start lease, call an admission service
   or operate on the current group instead of the requested ID.
2. If it returns ACCEPTED, ALREADY_TARGET or `ErrExecutionControlState`, make
   exactly one `SnapshotByID` call with the request context.
3. Classify the response using section 5. The snapshot's read transaction owns
   consistency and its cursor; no lifecycle lock is held through that read.
4. For other controller failures, use section 7 without a snapshot lookup that
   could mask the initiating failure.
5. Serialize the detached result. Never wait for a target status or poll inside
   the handler. The lifecycle, not this HTTP goroutine, owns completion.

A non-nil controller error takes precedence over any simultaneously returned
disposition. Before serializing a successful snapshot read, require its group ID
to match the requested ID; a mismatched/empty trusted result is 500 INTERNAL_ERROR,
never a response containing a different group's snapshot. Reuse the existing
projection; do not duplicate D2's full snapshot-validation business rules here.

If the accepted operation finishes, fails or is superseded before the snapshot
read, the body legitimately shows the newer committed state. An ACCEPTED result
still returns 202; do not upgrade it to 200 solely because its snapshot already
looks achieved. Conversely a 200 snapshot can reflect a later concurrent action
after ALREADY_TARGET was decided. HTTP status describes this submission; the
body describes the read's committed state. They are not one atomic transaction.

If a snapshot read fails after acceptance, return the classified read failure;
do not undo the action, repeat submission, invent a snapshot or use a stale cache.
The client must reconcile as described in section 9.

## 5. Exact disposition and state rules

| Controller result | Snapshot result | HTTP outcome |
| --- | --- | --- |
| ACCEPTED | Valid snapshot | 202, exact snapshot. Includes duplicates still pending and actions completed before the read. |
| ALREADY_TARGET | Valid snapshot | 200, exact snapshot; no extra action/events. |
| Control-state error for CANCEL | Exact target is durably CANCELLED | 200, exact snapshot. This covers retired completed groups; pending Cancel must instead return ACCEPTED under section 2. |
| Control-state error, any other saved state/action | Valid snapshot | 409 INVALID_EXECUTION_STATE with exact error/snapshot wrapper. |
| Control-state error | Target not found | 404 RESOURCE_NOT_FOUND, error only. |
| ACCEPTED or ALREADY_TARGET | Target not found | 500 INTERNAL_ERROR, error only; trusted dependency invariant failure. |
| Any of those read-required results | Read fails | Classified 500, error only. |
| Other controller error | Not read | Section 7; do not hide it behind terminal-state fallback. |

An ACCEPTED/ALREADY_TARGET result followed by a missing saved target is an
unexpected dependency invariant because history is retained. Report the internal
failure with operational context; do not re-create the group, claim the accepted
operation was undone, or fall back to `/runs/active`.

The saved-CANCELLED fallback is deliberately narrow. Saved PAUSED does not
justify accepting PAUSE when Resume is pending, and saved RUNNING does not
justify accepting RESUME when Pause or Cancel is pending. Non-terminal state
without matching live ownership cannot be resumed from SQLite.

| Live action state at ordering point | PAUSE | RESUME | CANCEL |
| --- | --- | --- | --- |
| Fully registered RUNNING, no pending control | 202 | 200 | 202 |
| Fully achieved PAUSED, no pending control | 200 | 202 | 202 |
| Pause pending | 202 duplicate | 409 | 202, supersedes Pause |
| Resume pending | 409 | 202 duplicate | 202, supersedes Resume |
| Cancel pending, including cleanup after CANCELLED commit | 409 | 409 | 202 duplicate |
| Retired CANCELLED | 409 | 409 | 200 |
| SUCCEEDED, FAILED or INTERRUPTED, no live work | 409 | 409 | 409 |
| Non-terminal record without matching registered live owner | 409 | 409 | 409 |
| Saved group absent | 404 | 404 | 404 |

Gate/stopping/fatal failures override this normal-state table. A valid group
whose initial worker registration is incomplete is a temporary 409, not permission
to control a partially established owner. A natural terminal outcome that beats
an accepted action is preserved; 202 does not promise that CANCELLED will replace
an already won SUCCEEDED/FAILED result.

## 6. Complete response examples

These are illustrative contract fixtures; section 12 records actual verification.
For readability they use one selected device; multi-device bodies have the same
shape and include all selected members. Cursors are illustrative Runtime-wide
watermarks, never per-action IDs or progress percentages.

### 6.1 Accepted Pause — HTTP 202

Delay is still running, so this response does not yet confirm PAUSED.

```json
{
  "cursor": "42",
  "runGroup": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RUNNING",
    "createdAt": "2026-11-23T08:00:00.000Z",
    "finishedAt": null,
    "summary": {
      "total": 1, "queued": 0, "running": 1, "paused": 0,
      "succeeded": 0, "failed": 0, "cancelled": 0, "interrupted": 0
    }
  },
  "deviceExecutions": [{
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "adbSerial": "emulator-5554",
    "status": "RUNNING",
    "nextNodeId": "delay",
    "createdAt": "2026-11-23T08:00:00.000Z",
    "startedAt": "2026-11-23T08:00:00.010Z",
    "finishedAt": null,
    "terminalError": null
  }]
}
```

### 6.2 Already paused — HTTP 200

Delay has committed and the retained worker is parked before Tap. Repeating
PAUSE has no additional effect. This is also the shape returned by a later GET.

```json
{
  "cursor": "47",
  "runGroup": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "PAUSED",
    "createdAt": "2026-11-23T08:00:00.000Z",
    "finishedAt": null,
    "summary": {
      "total": 1, "queued": 0, "running": 0, "paused": 1,
      "succeeded": 0, "failed": 0, "cancelled": 0, "interrupted": 0
    }
  },
  "deviceExecutions": [{
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "adbSerial": "emulator-5554",
    "status": "PAUSED",
    "nextNodeId": "tap",
    "createdAt": "2026-11-23T08:00:00.000Z",
    "startedAt": "2026-11-23T08:00:00.010Z",
    "finishedAt": null,
    "terminalError": null
  }]
}
```

### 6.3 Conflicting action — HTTP 409

RESUME is now pending, but SQLite still shows PAUSED. A new PAUSE conflicts;
that old persisted state cannot turn this request into an achieved duplicate.

```json
{
  "error": {
    "code": "INVALID_EXECUTION_STATE",
    "message": "action is not valid for the current execution state",
    "retryable": false
  },
  "snapshot": {
    "cursor": "47",
    "runGroup": {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "requestId": "550e8400-e29b-41d4-a716-446655440000",
      "status": "PAUSED",
      "createdAt": "2026-11-23T08:00:00.000Z",
      "finishedAt": null,
      "summary": {
        "total": 1, "queued": 0, "running": 0, "paused": 1,
        "succeeded": 0, "failed": 0, "cancelled": 0, "interrupted": 0
      }
    },
    "deviceExecutions": [{
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "adbSerial": "emulator-5554",
      "status": "PAUSED",
      "nextNodeId": "tap",
      "createdAt": "2026-11-23T08:00:00.000Z",
      "startedAt": "2026-11-23T08:00:00.010Z",
      "finishedAt": null,
      "terminalError": null
    }]
  }
}
```

RESUME and CANCEL success bodies use the same exact SnapshotResponse. Confirmed
Resume sets group RUNNING and surviving devices QUEUED, possibly already RUNNING
by the read. Confirmed cancellation sets surviving devices CANCELLED with null
nextNodeId and finishedAt; already terminal siblings retain their outcomes.
Do not add `actionId`, `accepted`, `pending`, `success` or `operation` fields.

## 7. Error mapping, timeouts and logging

| Condition | HTTP / code | Body and retryable |
| --- | --- | --- |
| Missing/invalid bearer | 401 UNAUTHORIZED | Existing error-only envelope; false. No dependency calls. |
| Invalid path/body/media/query | 400 INVALID_REQUEST | Existing error-only envelope; false. |
| More than 1,024 body bytes | 413 PAYLOAD_TOO_LARGE | Existing error-only envelope; false. |
| Exact target absent after control-state error | 404 RESOURCE_NOT_FOUND | Existing error-only envelope; false. |
| Ineligible transition with readable snapshot | 409 INVALID_EXECUTION_STATE | Exactly error plus snapshot; false. |
| Runtime stopping (`ErrExecutionStopping`) | 503 INTERNAL_ERROR | Error only, message `runtime is stopping`; false, matching current start API convention. |
| Persistence/commit-unknown/integrity failure | 500 PERSISTENCE_FAILED | Error only, message `unable to process workflow action` for control failure or existing snapshot-read message for read failure; false. |
| Missing dependency, unexpected disposition/error, missing target after success or mismatched snapshot identity | 500 INTERNAL_ERROR | Existing generic error-only envelope; false. |

Place the approved invalid-state code/message and action failure message in
shared constants. Use `errors.Is` through contextual wrappers. Snapshot error
classification should reuse the existing run-read mapping rather than drift
into a second set of persistence/not-found codes.

Use the request context for submission/read and existing HTTP read/write
timeouts. Do not detach the HTTP read from its cancelled request or introduce a
new long-poll timeout. If the client disconnects, normal response writing may be
impossible; do not manufacture a successful delivery claim. Owned action work
continues after acceptance under H9–H12's bounded rules.

Use `apiresponse.SetFailure` for the extended 409 envelope and existing
AbortWithCause for other failures. Keep one middleware request log; 4xx/5xx are
ERROR. Wrap internal failures with a fixed operation such as
`apply_run_action`/`read_action_snapshot`, retaining safe reason metadata. Never
log bearer headers, bodies, variables, outputs or protected workflows. H12's
async completion/failure log is a separate lifecycle event, not a second HTTP log.

## 8. Idempotence and concurrency boundaries

There is no action requestId or replay table. Duplicates are defined against
the group's current live operation/target, not against a historic button click.

- A repeat of the same pending action returns 202 without another coordinator,
  deadline extension, dispatch or synthetic event.
- A repeat of an achieved target returns 200 without writes. CANCEL on a
  retired CANCELLED group uses the exact saved record.
- A late PAUSE after an intervening Resume is a new Pause. Do not automatically
  resend old actions during reconnect or poll-loop recovery.
- Do not auto-retry mutating actions after a lost response. Read the exact group
  and let the user reconcile intent; a persisted RUNNING status cannot reveal
  every pending control.
- Simultaneous actions serialize under the existing lifecycle mutex; only those
  ordering points determine which pending request wins. HTTP arrival timestamps
  and snapshot read order are not action locks.
- A historic ID never redirects to a newer active group. Actions on prior runs
  cannot acquire authority over new RAM, workers or membership.

No new lock spans HTTP, SQLite and lifecycle. Snapshot reads remain consistent
read transactions, with complete selected membership and derived summaries.
Payload size grows with selection N; do not truncate responses to an old device
limit. Submission adds no per-node queries or device preflight.

## 9. Client integration deliverable and manual scenario

At H13 implementation, update the [shared managed Runtime guide](../integration/managed_runtime.md),
[Studio guide](../integration/studio.md) and [Launcher guide](../integration/launcher.md)
as one delivery. Keep one authoritative shared endpoint reference, with short
app-specific examples pointing to it. Remove contradictory claims that HTTP
Pause/Resume/Cancel is unavailable only after the route is verified.

The shared reference must include every action body, current bearer/header
acquisition, full 200/202/409 examples, other error codes, polling, disconnect
reconciliation and the distinction between Cancel and owner Stop and exit.
Provide Linux curl, Windows PowerShell and Postman usage. Use the actual
`apiBaseUrl` from readiness/discovery; it already ends in `/api/v1`.

Client sequence:

1. Use the existing owner-managed readiness/authentication flow before each
   operation; obtain current process credentials. Do not launch a second owner.
2. Store the runGroup ID returned by POST /runs and send an action to that ID.
3. For 202, show that action as pending locally; do not label a persisted status
   as its completed target until observation confirms it. Poll
   `GET {apiBaseUrl}/runs/{id}` at a recommended 500 ms while needed, one request
   in flight; stop on completion, terminal outcome, view disposal or owner exit.
4. For Pause, confirmation is PAUSED or a terminal winner. For Resume, RUNNING
   can include QUEUED members; a fast terminal outcome is also possible. For
   Cancel, watch for a terminal result, normally CANCELLED. A terminal SQLite
   snapshot does not separately prove that in-process cleanup has retired;
   repeat CANCEL can still be 202 while H12 finishes. Normal next-run handoff
   remains responsible for joining any residual owner cleanup.
5. On 409 render its snapshot, report the conflict and reconcile user intent.
   Do not loop the rejected action automatically. On a lost response/read
   failure, GET the same ID; do not infer that no action was accepted.
6. If Runtime has exited, readiness may relaunch it for an explicit operation,
   but D6 returns unfinished work as INTERRUPTED. Never RESUME an old run by
   replaying its POST /runs or rebuilding its RAM from SQLite.

Suggested real-API walkthrough after H12/H13 code exists: start a readable
Start → 10-second Delay → Tap → End fixture; submit PAUSE during Delay; observe
202 then PAUSED and stable attempt count; repeat PAUSE for 200; RESUME and observe
the same execution IDs; run again, Pause then Cancel; observe CANCELLED, retained
history and successful new admission without Runtime restart. Use controllable
fixtures to demonstrate opposing-pending 409 reliably; a fast manual click is
not deterministic evidence of that race.

Studio integrates current draft execution. Launcher may use the same group
controls for a supported run, but this phase does not deliver protected `.a2m`
admission, authorization or decryption; those remain Phase J. SSE is Phase I.

## 10. Implementation slices and test plan

| Slice | Work | Dependency |
| --- | --- | --- |
| 1 | DTO/constants, bounded decode, handler mapping and controller doubles. | Can be written against section 2 before H12 exists. New tests require scope approval. |
| 2 | Controller binding, constructor fixture adaptations, real route and generated Wire. | Real H12 Cancel must satisfy Controller; no stub to make production compile. |
| 3 | Real lifecycle/SQLite/API acceptance and shared/app-specific guide updates. | H12 verified with H9–H11; complete endpoint required. |

This is one H13 responsibility delivered in reviewable slices, not a renumbering
of the roadmap. Reuse existing API middleware, run handler and lifecycle fixtures.

| Test group | Required assertions |
| --- | --- |
| Strict transport | All three valid actions; missing/null/wrong/unknown values and keys; duplicate/escaped keys; trailing JSON; invalid UTF-8; malformed media/encoding/query/path; exact byte limit and chunked over-limit. Invalid requests call neither dependency. |
| Authentication/router | Correct POST route under existing bearer/no-store middleware; bad bearer beats body validation; no accidental conflict with active/by-ID routes; no redirects or new method behavior. |
| Mapping | Each row of section 5; exactly one method plus at most one exact-ID snapshot read; accepted action followed by advanced/terminal snapshot stays 202. |
| Duplicate/terminal | 200 achieved targets; pending duplicate 202; CANCELLED-but-cleaning still 202; retired CANCELLED 200; other terminal conflicts 409; no writes/events on achieved duplicates. |
| Pending conflicts | Resume during Pause and Pause during Resume give 409 even if persisted state looks achieved; Cancel supersedes both; no later dispatch from a superseded Resume. |
| Failure | Missing target 404; persistence/integrity/unknown-commit 500; stopping 503; unknown disposition/nil dependency 500; accepted action survives response-read failure and submission-context cancellation. |
| Existing run compatibility | Start/replay, by-ID/active, auth, artifacts and constructor-adapted fixtures retain their exact response behavior. |
| Real integration | Real SQLite plus controlled lifecycle handlers prove pause/duplicate/resume/cancel, fresh next run, correct retained history/RAM lifetime and stop joining. HTTP mocks alone do not close this gate. |
| Platforms/evidence | Linux run and race checks; Windows agent/affected-test cross-build reported separately; native Windows/live Android acceptance claimed only when actually exercised. |

Run focused tests and race checks, existing API/process regressions, build, vet,
Wire generation and whitespace checks. Report the missing G22 Python harness as
a baseline regression gap if still present. Do not count it as passed or add
unrelated repairs to H13. H14/H15 own wider crash and independent-device acceptance.

## 11. Completion checklist

H13 is complete only when all of the following are true:

- The single authenticated route supports PAUSE, RESUME and real H12 CANCEL.
- Strict DTO validation, classified errors and exact snapshot/wrapper shapes
  match this spec and A7; pending state never gets guessed from SQLite alone.
- No HTTP operation waits for workflow completion, publishes an uncommitted
  state, restarts workers or bypasses owner shutdown.
- Real lifecycle/SQLite integration proves controls and subsequent runs; tests
  clearly separate controller-double evidence from production implementation.
- Shared, Studio and Launcher guides describe the verified endpoint, credentials,
  polling, conflicts and process-management distinction with runnable examples.
- Current roadmap/status records show the actual checks and outstanding native
  Windows/real-device evidence. H12 absence is never labeled a finished H13.

H6–H8 Wait Variable behavior remains explicitly deferred. Wider host/process/live
acceptance remains H14/H15; the implementation evidence below does not close it.

## 12. Implementation and verification evidence — 2026-11-23

`internal/api/runs/actions.go` implements the Controller interface, Action
handler, bounded strict decoder and classified response mapping. DTOs remain
in `dto.go`, action/error/body-limit constants in `pkg/constants/api.go`, and the
single route lives inside the existing `/api/v1` router group. Shared JSON media
validation is reused by Start and Action. Snapshot read classification is reused;
it now also classifies commit-unknown as persistence failure.

The constructor takes reader/starter/controller explicitly. Wire binds the
existing lifecycle singleton to `runs.Controller`; generated Wire was regenerated.
Existing handler/API fixtures were adapted to the constructor. No engine,
lifecycle, database schema, migration or dependency changed for H13.

| Evidence | Executed result |
| --- | --- |
| `internal/api/runs/actions_test.go` | Strict JSON/duplicate/UTF-8/media/path/query/auth/body-size checks; exact byte boundary and unknown-length overflow; disposition, terminal fallback, error precedence, missing/mismatched target, current request context and no resubmission. |
| `internal/api/actions_test.go` | Real two-device runner/checkpoints/SQLite through the actual router: pending duplicate/opposite conflict, Pause, Resume, Cancel, retired duplicate, fresh run, saved non-live conflict, auth and contextual ERROR logging without tokens. |
| Existing built-agent H4 harness | Extended with real H13 HTTP Pause/Resume/Cancel, 409 terminal conflict, joined duplicate acknowledgement and a fresh successful group; production composition and private owner/session protocol, controlled ADB endpoints. |
| Integration guides | One [shared action reference](../integration/run_actions.md) with complete response examples, Linux curl, PowerShell 7, Postman and process/reconciliation rules; linked from managed/Studio/Launcher guides. |

Executed focused repeated race checks:

```bash
go test -race ./internal/api ./internal/api/runs -run 'TestRunAction|TestAction' -count=3 -timeout=60s
go test -race ./... -count=1
go test -race ./cmd/agent -skip '^TestG22CombinedEngineAcceptance$' -count=1 -v
make wire
make fmt
make build
go vet ./...
git diff --check
```

Focused checks pass. The full race command passes all packages except the known
G22 test whose `cmd/agent/testdata/g22_acceptance.py` fixture is absent. The
separate available cmd/agent run passes, including the actual-agent H13 extension.
Do not report this as an unqualified full-suite pass. Windows amd64 agent and
affected API test binaries cross-compile; native Windows and live Android H13
acceptance were not executed. Linux shell examples are syntax-checked; real HTTP
behavior is verified through the managed-agent harness, not a claim that every
client UI or PowerShell example has been run.
