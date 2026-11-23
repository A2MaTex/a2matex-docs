Understood. No implementation or reference documents yet. I only analyzed the existing workspace.

> **Historical analysis:** Phase A now supersedes the device-identity and
> provider-planning sections below. The accepted MVP uses an in-memory session
> inventory selected by current ADB serial, with no persistent Device entity.

## Product understanding

The Runtime Agent is a local, per-user Go daemon shared by Studio and Launcher:

- Studio uses Development mode with draft Runtime IR.
- Launcher uses Production mode with an installed release and authorization.
- Runtime owns session device discovery, execution state, checkpoints, events, artifacts, and detailed local history.
- Studio and Launcher are clients; they must not call ADB directly or reconstruct execution state.
- Execution runs in one long-lived agent process, with one isolated `DeviceExecution` per selected device. Capacity equals admitted membership, with no fixed count ceiling or separate maximum-device setting. H4 still owns public multi-device dispatch.

The functional scope is well described in [the 69 Runtime requirements](/home/dino/dev/a2matex/a2matex-plan/specifications/3.1.4_3.2.5_a2matex_runtime_functional_requirements.md:100), but it is not yet strict enough for implementation.

## Current readiness assessment

The product concept is strong, but the contracts are not implementation-ready.

The largest blocker is exactly what you identified: strict schemas are missing.

### 1. Runtime IR has no authoritative schema

The workspace only contains illustrative examples:

```json
{
  "irVersion": "1.0",
  "entryNodeId": "start_01",
  "nodes": []
}
```

There is no strict definition for:

- allowed root fields;
- node representation;
- handler identity and version;
- bindings and expressions;
- transitions;
- retry and timeout policy;
- controlled loops;
- required capabilities;
- workflow inputs;
- extension and compatibility rules;
- maximum sizes and counts.

The Backend currently accepts `runtimeIr` only as a generic JSON object. That is insufficient for a strict Go runtime.

### 2. Node handler versioning is inconsistent

Runtime documentation expects:

```text
node type + handler version → registered Go handler
```

But the current Backend Node Catalog has:

- `type`
- `runtimeHandler`
- `minimumRuntimeVersion`
- definition `contentHash`

It does not define an explicit handler contract version. Meanwhile Studio documentation shows `WorkflowNode.version`.

We must decide whether version belongs to:

- the node instance;
- the Node Definition;
- the runtime handler;
- or a separate node-contract version.

### 3. Planned nodes do not match the current catalog

The current Backend catalog contains 11 nodes:

- `flow.start`
- `flow.end`
- `flow.delay`
- `utility.log`
- `android.launch_app`
- `android.tap`
- `android.swipe`
- `android.input_text`
- `android.press_back`
- `android.press_home`
- `android.screenshot`

Runtime and Studio documents additionally mention:

- If/Switch
- Repeat
- Wait For Element
- Long Press
- selector-based Tap
- HTTP
- Telegram
- Google Sheets

Those cannot be included in the first Runtime implementation until their Node Definition and Runtime IR contracts exist.

### 4. Android action contracts conflict

The current catalog defines Tap using absolute integer `x` and `y`.

Runtime planning also describes:

- normalized coordinates;
- selectors;
- UIAutomator lookup;
- resource ID, text, content description, and class selectors.

Those are materially different contracts. The first Runtime must implement only the approved catalog contract unless the catalog is revised first.

### 5. Local API schemas are missing

Endpoint names exist, but strict contracts do not:

- request and response envelopes;
- local session-token creation and delivery;
- random-port discovery;
- handshake compatibility rules;
- error codes and field paths;
- event transport and reconnection cursor;
- idempotency for start/pause/resume/cancel;
- API version compatibility.

### 6. Session device selection is decided

Runtime keeps only the current ADB discovery result in memory. Clients select
opaque ADB serials, Runtime rejects duplicates, and every selected serial is
revalidated before run admission. A5 copies the accepted serial into the
corresponding `DeviceExecution`. There is no persistent Device UUID, provider
correlation, alias, inventory history, or device table in the MVP.

### 7. Execution state needs transition tables

States are listed, but allowed transitions and invariants are not fully defined.

For example:

- whether a queued run group can be paused;
- when a run group becomes `FAILED` versus partially successful;
- whether resume continues at the current or next node;
- how offline-device recovery works;
- whether cancelling one device cancels its siblings;
- how `PAUSED` with `AUTHORIZATION_EXPIRED` participates in group status.

### 8. Persistence is currently conceptual

SQLite entities are listed, but strict schemas are missing:

- column types and constraints;
- status checks;
- unique execution sequences;
- node attempt identity;
- transaction boundaries;
- event cursor semantics;
- artifact paths and checksums;
- retention and cleanup;
- schema migration/versioning;
- sanitized versus protected fields.

### 9. Development and Production should be separated

Development mode can be built without Backend module group 3.

Production mode depends on future contracts for:

- `.a2m` package manifest;
- Product Release identity;
- package/payload hashes;
- signed execution leases and renewal;
- Runtime session key pairs and key envelopes;
- package-required capability compatibility;
- authorization expiry;
- workflow encryption.

Production security should therefore be a later adapter around the same execution core, not mixed into the first modules.

## Proposed decomposition model

Before implementation, the Runtime should be divided into contract modules first:

1. Shared identifiers, versions, enums, timestamps, and error schema.
2. Runtime IR schema.
3. Node handler contract and registry schema.
4. Device and discovery schema.
5. Development execution request schema.
6. Run group, device execution, and node execution schemas.
7. State-transition specification.
8. Execution event and reconnection schema.
9. SQLite persistence schema.
10. Local API, authentication, and handshake schema.
11. Typed device-command and result schema.
12. Artifact metadata schema.
13. Production package and authorization schemas—deferred.

Only after those are approved should implementation be divided into small modules such as process foundation, command runner, LDPlayer parsing, ADB parsing, device correlation, registry, typed commands, handler registry, single-device engine, persistence, events, scheduler, and lifecycle control.

Each module should have its own scope, dependencies, strict input/output schemas, invariants, tests, and checkout criteria.

## Conclusion

The Runtime Agent is not ready for implementation yet. The first item to resolve is the canonical Runtime IR and node-handler version contract, because nearly every later module depends on it.

No workspace files were changed.

# Modules

For the Runtime Agent product, we should separate strict contracts, core engines, infrastructure modules, handlers, and external integrations.

This is an analysis breakdown only—no documents or code yet.

## 1. Contract and schema modules

These must be defined before implementation.

1. **Runtime IR Schema**
   - Workflow format version.
   - Nodes, transitions, bindings, inputs, capabilities.
   - Timeout, retry, and error policies.
   - Strict unknown-field rejection and size limits.

2. **Node Contract Schema**
   - Node type and contract version.
   - Handler identity/version.
   - Config, input, and output schemas.
   - Output ports and required capabilities.

3. **Session Device Selection Rules**
   - Current ADB discovery only.
   - Selection by opaque ADB serial.
   - Duplicate rejection and start-time readiness revalidation.
   - No persistent Device entity or cross-session correlation.

4. **Execution Request Schema**
   - Development versus Production request.
   - Selected devices.
   - Per-device inputs.
   - Maximum concurrency and failure policy.
   - Credential references and authorization.

5. **Execution State Schemas**
   - Run group.
   - Device execution.
   - Node execution and attempts.
   - Checkpoint.

6. **Event Schema**
   - Event type and sequence.
   - Run group, device execution, device, node, and attempt identity.
   - Sanitized payload.
   - Reconnection cursor.

7. **Error Schema**
   - Stable error code.
   - Failure category and boundary.
   - Safe message and structured field violations.
   - Retryability.

8. **Persistence Schema**
   - SQLite tables, keys, constraints, indexes, and migrations.
   - Retention and cleanup rules.

9. **Local API Schema**
   - Requests, responses, authentication, and error envelopes.
   - Handshake and compatibility response.
   - Event-stream protocol.

10. **Typed Device Command Schema**
    - Tap, Swipe, Input Text, Launch App, key presses, and Screenshot.
    - Command result and confirmation level.

## 2. Runtime foundation modules

1. **Agent Supervisor**
   - Startup and graceful shutdown.
   - Component lifecycle.
   - Root cancellation context.

2. **Configuration Module**
   - Runtime mode.
   - Tool locations.
   - Loopback address.
   - Timeouts, concurrency, database, artifacts, and logging.

3. **Version and Compatibility Module**
   - Runtime version.
   - Runtime IR versions.
   - Supported handler versions.
   - Client compatibility.

4. **Local Authentication Module**
   - Local session token.
   - Constant-time token verification.
   - Token rotation and safe delivery to Studio/Launcher.

5. **Local API Transport**
   - Loopback HTTP server.
   - Request validation.
   - Health, runtime information, device, and execution endpoints.

6. **Realtime Transport**
   - WebSocket or SSE.
   - Authentication.
   - Reconnection from a sequence cursor.

7. **Safe Logging and Redaction**
   - Structured logs.
   - Secret and protected-payload removal.
   - Run/device/node correlation fields.

## 3. Device modules

1. **Toolchain Locator**
   - Validate the LDPlayer installation.
   - Resolve `ldconsole.exe`, `ld.exe`, and bundled `adb.exe`.

2. **Process Executor**
   - Execute structured executable arguments.
   - Capture stdout, stderr, exit code, and duration.
   - Enforce timeout and cancellation.
   - Never invoke publisher-controlled text through a shell.

3. **LDPlayer Discovery**
   - Execute and parse `ldconsole list2`.
   - Normalize instance state.

4. **ADB Discovery**
   - Execute and parse `adb devices -l`.
   - Represent online, offline, and unauthorized endpoints.

5. **Device Correlator**
   - Associate LDPlayer instances with their correct ADB endpoints.
   - Handle incomplete, ambiguous, and changing mappings.

6. **Device Registry**
   - Maintain authoritative device inventory.
   - Detect state changes.
   - Expose snapshots to clients.

7. **Device Eligibility Policy**
   - Determine whether a device is online, supported, command-ready, and available.

8. **Device Lock Manager**
   - Exclusive execution ownership per device.
   - Safe acquire/release behavior.

9. **Device Controller**
   - Common interface for device operations.
   - Select ADB or LDPlayer-specific adapters.

10. **ADB/LDPlayer Adapters**
    - Convert typed commands into safe executable arguments.
    - Bind every command to the correct device.

## 4. Core execution engines

These are the actual decision-making engines.

### Runtime IR Validation Engine

Validates:

- format version;
- graph structure;
- node and handler compatibility;
- transitions;
- inputs and bindings;
- capabilities;
- limits.

### Preflight Engine

Validates the complete run before creating active executions:

- request;
- workflow;
- devices;
- per-device inputs;
- concurrency;
- capabilities;
- Development/Production policy.

No device should begin if run-level preflight fails.

### Input and Binding Resolution Engine

Resolves node input from:

- static configuration;
- workflow inputs;
- per-device inputs;
- variables;
- previous node outputs.

Workflow inputs, node outputs, and Flow variables remain isolated by
`DeviceExecution`. Global variables are explicitly shared within one RunGroup.

### Workflow Interpretation Engine

For one device execution:

```text
Current node
→ resolve input
→ execute handler
→ validate output
→ select transition
→ checkpoint
→ continue
```

Nodes remain sequential within one device.

### Transition Engine

Handles:

- success and error ports;
- conditional branches;
- controlled loops;
- timeout paths;
- retry exhaustion;
- terminal nodes.

### Retry and Timeout Engine

Determines:

- allowed attempts;
- retry delay;
- retryable categories;
- timeout behavior;
- safe idempotency-key reuse.

### Handler Registry

Maps:

```text
node type + contract version → registered handler
```

Unknown or incompatible handlers must be rejected during preflight.

### Typed Command Engine

Accepts only typed commands from registered handlers and delegates them to the selected device adapter.

It must distinguish:

- `DISPATCHED`
- `OBSERVED`
- `CONFIRMED`

### Single-Device Execution Engine

Owns one independent `DeviceExecution`:

- current node;
- variables;
- node outputs;
- sequence;
- retries;
- cancellation and pause boundary;
- terminal result.

### Scoped Variable Store

G16 owns RAM-only Flow maps per DeviceExecution and one shared Global map per
RunGroup in the Runtime process. Keep the same store through live Pause/Resume.
Release it after terminal settlement and joined workers; exit/crash loses it.
There is no variable table, migration, snapshot, or restart restoration.
G18 confirms the existing node checkpoint before publishing an assignment to
RAM and allowing readers/continuation. See the
[G16 contract](g_workflow_engine/g16_scoped_variable_storage.md).

### Multi-Device Scheduling Engine

Creates one device execution per selected device and provides:

- bounded concurrency;
- queueing;
- goroutine supervision;
- failure isolation;
- device locking.

### Run Group Coordinator

Owns:

- group lifecycle;
- execution collection;
- counters;
- failure policy;
- final aggregate result.

### Lifecycle Control Engine

Handles:

- pause;
- resume;
- cancel;
- device-offline pause;
- authorization pause.

### Capability Compatibility Engine

Checks each handler against:

- workflow-declared capabilities;
- Development-mode policy;
- Runtime-supported capabilities.

## 5. Persistence and recovery modules

1. **SQLite Migration Module**
2. **Run Group Repository**
3. **Device Execution Repository**
4. **Node Execution Repository**
5. **Execution Event Store**
6. **Device Snapshot Store**
7. **Artifact Metadata Store**
8. **Checkpoint Engine**
   - Atomically saves result, next node, sequence, state, and event.
   - Scoped variable maps/values are not persisted; G18 publishes RAM after confirmation.
9. **Crash Recovery**
   - Marks previously unfinished work, including paused runs, `INTERRUPTED`.
   - Never resumes old runs, restores variables, or blindly replays side effects.
10. **Retention and Cleanup**
    - Execution history.
    - Old events.
    - Artifact files.
    - Interrupted or abandoned runs.

## 6. Event and monitoring modules

1. **Event Factory**
2. **Durable Event Store**
3. **In-Process Event Bus**
4. **Event Publisher**
5. **Execution State Query Service**
6. **Reconnect/Cursor Service**
7. **Run and Per-Device History Service**

Events must be persisted before being broadcast.

## 7. Initial node-handler modules

The first implementation should match the current Backend catalog exactly.

### Flow and utility handlers

- Start
- End
- Delay
- Log

### Android handlers

- Launch App
- Tap with absolute coordinates
- Swipe
- Input Text
- Press Back
- Press Home
- Screenshot

Handlers such as If, Repeat, Wait For Element, selector-based Tap, Long Press, HTTP, Telegram, and Google Sheets remain pending until their strict catalog and Runtime IR contracts exist.

## 8. Artifact and observation modules

These come after basic command execution.

1. **Artifact Store**
   - Screenshot and diagnostic files outside SQLite.
2. **Screenshot Processor**
3. **UI Hierarchy Collector**
4. **Element Locator**
5. **Current Activity Observer**
6. **Polling/Observation Engine**
7. **Artifact Cleanup**

Image matching and OCR should be later independent modules.

## 9. Connector modules

Deferred until Android execution is stable:

1. Connector Registry
2. Connector Worker Pool
3. Credential Reference Resolver
4. HTTP Connector
5. Telegram Connector
6. Google Sheets Connector
7. Connector Idempotency
8. Secret Redaction

Connectors remain request/response operations in the MVP—not fire-and-forget jobs.

## 10. Production-only modules

These depend on future Backend and Launcher contracts:

1. Package Manifest Parser
2. Package Integrity Verifier
3. Signature Verifier
4. Secure Package Loader
5. Execution Authorization Validator
6. Capability Grant Loader
7. Lease Expiration Monitor
8. In-Memory Workflow Decryption
9. Execution Summary Synchronizer
10. Sensitive Memory and Temporary Resource Cleanup

## Recommended implementation sequence

```text
Strict schemas
→ Runtime foundation
→ Safe process executor
→ LDPlayer discovery
→ ADB discovery
→ Device correlation and registry
→ Typed device commands
→ Device adapters
→ Runtime IR validator and Handler Registry
→ Flow handlers
→ Single-device engine
→ Android handlers
→ SQLite and atomic checkpoints
→ Durable events and state queries
→ Multi-device scheduler
→ Pause/resume/cancel and crash recovery
→ Observation and artifacts
→ Connectors
→ Production package/license integration
```

The most important boundary is that we should not call everything an “engine.” Parsing, persistence, transport, discovery, and storage should remain modules. Engines should be reserved for components that make execution decisions: validation, preflight, binding resolution, workflow interpretation, transitions, policy, scheduling, lifecycle, and recovery.

AGENTS.md loaded.
