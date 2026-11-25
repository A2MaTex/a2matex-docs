# Launcher Runtime integration

## Overview

Launcher starts Runtime as a native child when a product operation needs it.
The implemented lifecycle covers startup, authenticated readiness, conflicting
ownership, and idle shutdown. Production package execution remains pending. H4 provides a Development
multi-device start API; its owned run can be cancelled by process shutdown.

```mermaid
flowchart LR
    P[Product operation needs Runtime] --> C[Native Runtime client]
    C -->|Store locked| B[Show Runtime already in use]
    C -->|Ready and authenticated| R[Runtime session available]
    R -.-> X[Production run request pending]
    R --> S[Explicit idle shutdown]
```

## Start and check the session

### Planned Phase J handoff

The exact future Product API agreement is
[J1 — Production contracts](../j_protected_execution/j1_production_contracts.md).
Follow [the ordered Runtime/Backend/Launcher/manual plan](../../../phase_j_implementation.md)
and the replacement [J1–J16 index](../phases/phase_j.md). J1 is a specification
and public test-vector delivery, not a runnable Production build.

The planned order is installed package download/hash verification → J4 public
schema read/form → customer Run → J10 session preparation → Backend authorization
→ J11 protected start → J12 renewal/J13 Product state and SSE → J14 fresh-session
reauthorization if needed. Do not generate keys or start a lease merely to show
the form. Reuse the exact package identity when submitting its input values.

Runtime local Bearer, Backend customer Bearer and signed execution leases have
different recipients. Backend successes have `data/statusCode`; Runtime bodies
are direct DTOs. Preserve operation IDs on uncertain responses. J1 fixes these
types and the separate `/product-runs/*` / `/product-events` summary-only surfaces;
Development history/events/artifact routes must not disclose Product internals.
Production package/schema/API examples become integration instructions only
after their owning Runtime and Backend tasks are implemented and verified.

### Planned package installation and location contract

**Handoff requirement, not an implemented package API.** Follow
[J1 section 3](../j_protected_execution/j1_production_contracts.md#3-installed-package-identity-and-public-schema)
when implementing the installation/form and Production client tasks.

Launcher owns the selected package root. Default to the host's actual Documents
folder plus `A2MaTeX/launcher/product-packages`, allowing a user-selected absolute
folder later. Resolve relocated/localized Documents through OS APIs. Do not use
Runtime's SQLite state directory or assume the executable's folder.

```text
<selected package root>/
└── <productId>/
    └── <installedReleaseId>/
        └── <productSlug>-<releaseVersion>.a2m
```

The Backend customer download response must include `fileName`, sourced from
`Static.originalFileName`, alongside package identity/hash, size, URL and expiry.
Save that exact filename; never reconstruct it from a product's current slug or
extract it from the R2 URL. R2 storage naming is unchanged. Invalid/nonportable
server filenames fail installation rather than being silently renamed.

Keep one installation record per Product in the Launcher's existing local storage:
`productId`, `installedReleaseId`, `fileName`, verified `packageHash`. Store the
selected root in Launcher settings. No new Runtime installation table is needed.
The record chooses the package to run; filenames and UUID ordering do not choose it.

For schema reading and preparation, send the root as a header and selection as
JSON. This Linux example is illustrative until J4/J10 are implemented:

```http
Authorization: Bearer <current local Runtime token>
Content-Type: application/json
X-A2MaTeX-Package-Directory: %2Fhome%2Fdino%2FDocuments%2FA2MaTeX%2Flauncher%2Fproduct-packages
```

```json
{
  "packageRef": "11111111-1111-4111-8111-111111111111/22222222-2222-4222-8222-222222222222/auto-farming-1.0.0.a2m",
  "productId": "11111111-1111-4111-8111-111111111111",
  "releaseId": "22222222-2222-4222-8222-222222222222",
  "packageHash": "<verified SHA-256 from Backend download metadata>",
  "launcherVersion": "0.1.0"
}
```

The schema request uses this body directly; preparation wraps it in `package`
with its own `requestId` and `purpose`. Preserve the header for protected start
and fresh-session reauthorization; those bodies reference the prepared session,
not another package path. Other Product requests may receive the same header
through the common native adapter without using it to reload or redirect a run.
Never send it to Backend/R2 or add it to Studio's Development requests.

Encode the absolute native root once as RFC 3986 UTF-8 percent encoding (4 KiB
encoded maximum). Encode spaces as `%20`; `+` is not a space escape. For Windows,
`C:\Users\Dino\Documents\A2MaTeX\launcher\product-packages` becomes
`C%3A%5CUsers%5CDino%5CDocuments%5CA2MaTeX%5Clauncher%5Cproduct-packages`.
Keep `/` between the three JSON `packageRef` segments on both hosts; do not
percent-encode that JSON field. Runtime decodes the header once and rejects
malformed/repeated headers and unsafe paths. See the fixture bundle for both
host examples. This is not a generic local-file read API.

### Planned update flow: one installed Release per Product

Compare `installedReleaseId` with Backend's selected `currentRelease.id` when
checking for updates. Publishing alone does not change that pointer. Use eligible
customer product metadata; Publisher/Admin management APIs are not customer APIs.
The customer-download task must return trusted selected identity/hash/filename.
For owned products outside Marketplace discovery, its customer-facing projection
must also supply eligible current-release metadata rather than relying on discovery.

1. Request an eligible replacement download and stage it without deleting the
   installed package. Verify the expected hash and identity.
2. Wait until this Product has no preparing session or active/paused execution.
   Disable replacement while it is in use; an expired paused run still needs the
   same package for reauthorization.
3. Install the exact server filename in the replacement Release folder, switch
   the installation record, and discard the old package-bound input form/session.
   If switching fails, retain the old selected installation.
4. Remove the old Release folder after switching. Retry cleanup failures without
   reverting the installation record or deleting unrelated files.

Temporary staging/old files during replacement are expected; permanent local
multi-version retention, rollback and a customer version picker are outside MVP.
Backend immutable Release history and existing Runtime result retention remain
independent of package cleanup. A settings change affects future package
operations, not a pending session or live run. Do not move/remove its files until
that use ends. Directory changes require rebuilding the next package-bound form
and preparation, rather than changing a frozen root on a start retry.

### Existing native process integration

Browsing installed products can leave Runtime stopped. When an operation
requires it, call the native Runtime client described in the
[managed lifecycle guide](managed_runtime.md). The currently available
integration check is the authenticated `GET {apiBaseUrl}/runtime-info` request.
Apply the same readiness and replacement rules to every later operation.

If Studio owns the store, the competing managed launch exits with code `2`.
Show **Runtime already in use** and let the user release Runtime in the owning
app before trying again. Launcher keeps its own credentials for the process it
starts.

## Stop the owned process

Expose **Shut down Runtime** while Launcher owns the idle process. Send the
control command, wait for exit, and keep Launcher open. If Launcher itself
closes, stop the owned child as part of app shutdown.

The production run example depends on the verified package admission API.
Add the exact package request, run status, and **Stop and exit / Stay open**
flow once production execution is verified. H6–H8 variable waiting remains deferred. The Studio plaintext
authoring endpoints remain scoped to authoring.

## Current API inventory

| Route | Launcher use |
|---|---|
| `GET /api/v1/runtime-info` | Authenticate to the owned Runtime, verify readiness for each operation, and read compatibility information. |
| `GET /api/v1/health` | Optional troubleshooting check for HTTP liveness. A successful health response cannot release waiting operations. |
| `GET /api/v1/devices` | Read the last successful device inventory in the current session. |
| `POST /api/v1/devices/refresh` | Scan ADB and return the updated device inventory. |
| `GET /api/v1/runs/active` | Read the saved non-terminal RunGroup or receive 404 when idle. |
| `GET /api/v1/runs/{runGroupId}` | Read the requested saved progress or terminal result. |
| `GET /api/v1/device-executions/{deviceExecutionId}/attempts` | Read paged saved node attempts and committed outputs through [I1](managed_runtime.md#node-attempt-history-api--i1). |
| `GET /api/v1/events` | Follow committed Runtime-wide state events through [I4](managed_runtime.md#execution-event-stream--i4), with Bearer, Accept and optional Last-Event-ID. This does not add Production package admission. |
| `POST /api/v1/runs/{runGroupId}/actions` | Pause, Resume or Cancel an existing group; use the [shared action guide](run_actions.md). This does not supply Production package admission. |

Use the [shared API reference](managed_runtime.md#shared-api-reference) for
request headers, response examples, and error codes. A Launcher smoke test
starts its managed child, reads the private ready record, successfully probes
Runtime Info, sends the private shutdown command, and waits for process exit.
A competing-launch test should show **Runtime already in use** and leave the
first owner's process running.

Launcher has snapshot read routes. The current build also has a Development
[POST /runs route](managed_runtime.md#workflow-start-api), but no Production
package admission/start route. H13 group controls are documented in the
[shared action guide](run_actions.md). Do not send customer product
content through the plaintext Development API. Document those remaining
calls here as they become available, including package
input, authorization and compatibility, run identity, status and event
consumption, retry behavior, and the active-run shutdown flow. Plaintext
workflow validation and compilation belong to the [Studio guide](studio.md).

## Android device picker

When the customer opens device selection for a product, refresh the list
through Launcher's owned Runtime session. The cached read is available for
redisplaying the last successful result in that session. Provide a
**Refresh devices** action and keep Launcher ownership handling in the
native client. See the [shared E7 requests and responses](managed_runtime.md#device-inventory-api).

Use `adbSerial` as the selection key. Development tests can select one or more
READY devices; Production selection follows its later package-admission contract. Enable selection for `READY` entries,
show reasons for unavailable entries, and flag any selection lost after a
successful refresh. If refresh fails, show the failure even when the previous
result is still visible. Inventory calls require the local session token.
Product package authorization and the final live readiness recheck belong
to the future production run request.

## Read progress and retained results

Use the [shared run snapshot contract](managed_runtime.md#run-snapshot-apis) for
the exact response, errors, auth headers, Linux/Windows/Postman requests, and
[saved-result demo](managed_runtime.md#optional-saved-result-fixture-without-android).
These reads require no ADB call and are available before Production package
execution is integrated. They do not implement package authorization or start
execution from a saved record.

Use active to populate a current-run view. A 404 means idle, not a Runtime
readiness failure. Preserve a known RunGroup ID so by-ID can display its result
after it finishes or when another run is active. Read the same ID with the
current owned session after reconnecting; never reconstruct or repeat a start
from the returned nextNodeId. Store identity matters: a run in a different
state directory is not present in this database.

Use the native readiness gate before each request, pass the current session
token, and send no body or query. Show the group status/summary and ordered
device states with safe terminal errors. Keep the last displayed result separate
from a read failure. Stop automatic polling for a confirmed terminal run or a
closed view, with at most one outstanding poll per view. Cancelling a GET only
cancels that read; it does not stop the workflow or the owned Runtime process.

Run controls and owner-process shutdown remain distinct. Use the existing
private owner channel for shutting down Runtime, not these GET endpoints.
H4 owns the current multi-device Development start flow; Production package
authorization and its future Launcher start endpoint remain separate work. Snapshot API/router and built-agent checks
pass on Linux and cross-compile for Windows; native Windows execution remains
unverified.


## Development API and Production boundary

The Studio developer owns the current plaintext Test Run request. The
[shared start reference](managed_runtime.md#workflow-start-api) documents
its exact DTO, auth, 201/200 responses, identity replay, errors, and manual
examples for both client developers. The Launcher Production API will accept
an authorized package/path under Phase J; G14 does not supply that authorization.

For lifecycle integration, the whole admitted run group belongs to the Runtime owner,
not its HTTP connection. Private shutdown fences dispatch and commits G11
cancellation before storage closes; owner loss uses D6 recovery. Expose
**Stop and exit / Stay open** for any active execution owned by Launcher once
its Production start path is delivered. H13 HTTP Pause/Resume/Cancel is now
implemented; follow the [shared guide](run_actions.md) for bearer, body, 200/202/409
responses, polling and lost-response handling. A Cancel button keeps Runtime
alive; Shut down Runtime and app exit still use the private owner channel.

H4 exposes the H1–H3 coordinator through the existing Development route.
Use exact ordered serial/input pairs and highlight indexed unavailableDevices
diagnostics. This route does not authorize a customer Product package. Follow the [G14 expansion checklist](../g_workflow_engine/g14_start_run_api.md#15-phase-h-expansion-checklist).

## Device selection and execution capacity

Customers control resource usage by selecting devices for each run. There is no
separate **Maximum simultaneous devices** setting. Send the exact selected ADB
serials and corresponding inputs; Runtime validates syntax, uniqueness and fresh
readiness, then admits or rejects the complete selection. Do not silently filter
unavailable entries or deduplicate entries carrying different inputs.

Capacity equals admitted membership. The owner does not write capacity YAML,
inject a capacity environment variable or restart Runtime when the next run's
selection changes. There is no per-request concurrency field or settings API.
Hardware determines practical capacity; actual byte/schema/deadline failures
still apply independently of device count.

H4 public multi-device Development starts are implemented. Use the
[H4 walkthrough](../../../../a2matex-runtime/examples/h4/README.md) for developer
integration tests, while keeping customer Product execution disabled until the
Production package/authorization API is delivered. The
[shared API reference](managed_runtime.md#workflow-start-api) documents exact
requests, availability diagnostics, replay and complete snapshots.

The [combined H15 acceptance pack](../../../../a2matex-runtime/examples/h15/README.md)
verifies the existing Development routes, complete per-device results, artifacts
and group controls under Linux controlled ADB. Use it for developer integration
checks; live independent Android/native Windows acceptance remains pending. This
does not introduce or authorize the Phase J protected Product-package API.

## Saved node-attempt history — I1

The [shared history reference](managed_runtime.md#node-attempt-history-api--i1)
defines exact JSON, pagination, errors, Linux/Windows requests and active refresh.
Use the DeviceExecution ID from the selected run snapshot, the current native-owned
session and actual readiness gate. This read needs no connected Android target.
I1 supplies screenshot IDs for the existing artifact-download API; do not query
SQLite. Same-sequence status updates require refreshing an overlapping window,
not treating attempt pagination as an event cursor. Runtime history is
Linux-verified; native Windows and Launcher UI acceptance remain separate.
This adds no protected `.a2m` admission or authorization; those remain Phase J.

## Continuous execution events — I4

Use the [shared SSE reference](managed_runtime.md#execution-event-stream--i4)
for exact headers, framing, errors, Linux/Windows/Postman requests and verified
limits. Open one connection through the native owner client after actual
readiness. Send the current Bearer and `Accept: text/event-stream`; no query or
body is accepted. Keep credentials in native memory and forward only safe
decoded event data to the UI.

Filter by the selected run/device correlation IDs. Save the event cursor as a
string only after applying its data. Heartbeats are comments, not progress.
Reconnect after that processed cursor; on initial410 repair the relevant views
from authoritative snapshots and reconnect after the repaired cursor. A broken
stream is not proof that a workflow failed or that Runtime died: probe the
owned session and use the existing replacement policy only on actual readiness
failure. Explicit owner shutdown suppresses automatic reconnect/relaunch.

I4 is implemented and Linux-verified with HTTP/SQLite and managed-agent checks.
Native Windows and Launcher UI acceptance remain separate. This event API can
observe retained/admitted runs but does not authorize decrypting or starting a
customer `.a2m` package through Development endpoints.
