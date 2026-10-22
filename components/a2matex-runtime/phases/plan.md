The Runtime Agent should be implemented as small, reviewable modules with strict contracts first. The core Runtime can be completed without the Submission module; only the Production package/license path depends on the later Product–Submission–Release backend group.

This reorganized plan also corrects the current LDPlayer-first and handler-version assumptions in the existing Runtime documents.

## 1. Product overview — how the Runtime Agent works

A2MaTeX lets a user design an Android automation visually, but the workflow
still needs a trusted component on the user's computer to turn that design into
real device operations. The Runtime Agent is that component. It is the local
execution bridge between Studio or Launcher and any Android endpoint that ADB
can detect, including physical devices and emulators. ADB, the Android Debug
Bridge, is the standard tool used to communicate with those devices.

The MVP is designed around a simple promise:

> Design the workflow once, select the available Android devices, and let the
> Runtime execute the same validated steps on each device while preserving a
> trustworthy record of what happened.

### The user journey

1. The user builds a workflow from the supported nodes in Studio.
2. Studio asks the local Runtime to validate the graph and shows deterministic
   errors before the user starts a run.
3. The client sends the exact executable Runtime IR—the machine-readable
   workflow package—together with per-device inputs and selected current ADB
   serials through the authenticated loopback API.
4. The Runtime independently validates the executable document and rechecks
   every selected device. Invalid input creates no run and sends no device
   command.
5. The Runtime creates a persisted run group, schedules only the configured
   number of devices concurrently, and executes nodes sequentially on each
   device.
6. A node handler converts each approved workflow operation into one of the
   Runtime's predefined device commands. Only the device subsystem may
   translate that command into a process invocation for the selected ADB
   serial.
7. After every attempt, the Runtime atomically records the result, validated
   output, next node, execution state, and client event before continuing.
8. Studio or Launcher follows progress through snapshots and replayable events,
   and can pause, resume, or cancel a run without directly controlling ADB.

```text
Studio (Development) or Launcher (Production)
                 |
                 | authenticated local API
                 v
       validate exact workflow and inputs
                 |
                 v
       persist RunGroup and device work
                 |
          bounded scheduler
          /       |       \
         v        v        v
     Device A  Device B  Device C
      active    active     queued
         |        |
         v        v
      sequential node execution
                 |
                 v
        persisted checkpoints
                 |
                 v
       snapshots + replayable events
                 |
                 v
         observable client status
```

### Why this architecture matters

- **Real execution:** MVP Android nodes must perform real ADB operations; a
  fake executor is only a test double and cannot satisfy product acceptance.
- **Device independence:** the execution path depends on the ADB protocol, not
  on LDPlayer-specific APIs, so the same model supports physical devices and
  ADB-visible emulators.
- **Safety at the boundary:** workflows cannot provide executable paths, shell
  fragments, raw ADB arguments, target serials, or artifact paths.
- **Predictable multi-device operation:** devices run independently under a
  configured concurrency limit, while nodes remain ordered within each device.
- **Observable and recoverable execution:** committed state is the source of
  truth; events describe committed changes, and a client can recover its view
  from a snapshot when event history is unavailable.
- **Failure isolation:** a disconnected or failed device does not silently
  corrupt another device's progress, and uncertain work is not automatically
  replayed after a crash.
- **Contract consistency:** Studio, Backend, and Runtime use the same accepted
  contract fixtures so disagreements can be detected before release.

Development mode connects Studio directly to the local Runtime for validation
and draft execution. Production mode later connects Launcher to the same core
engine after Backend package, entitlement, and release contracts exist. The
execution engine remains local in both modes; neither Studio nor Launcher owns
ADB commands or authoritative execution state.

This section describes the intended MVP behavior. Each capability becomes an
implemented claim only after its phase gate and real-device acceptance checks
pass.

## 2. MVP scope lock

The first usable Runtime release includes:

- Generic Android device discovery through ADB.
- Real execution—never simulated execution.
- The 11 node handlers already defined by Backend:
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
- Success and error transitions.
- Timeout, bounded retry, and error policies.
- Single-device and bounded multi-device execution.
- Persistent execution state, checkpoints, typed events, and screenshot artifacts, plus separate structured operational logs.
- Pause, resume, cancel, disconnect recovery, and crash interruption handling.
- Studio Development mode.
- Later, Launcher Production mode when the Backend release APIs exist.

Explicitly deferred until after release:

- Node-handler versioning.
- Conditional nodes, switch, loops, parallel/join graph logic.
- Selector-based interaction.
- Wait-for-element.
- OCR and image matching.
- HTTP, Telegram, Google Sheets, and other connectors.
- Test-one-node and arbitrary run-from-node modes until their binding and
  stopping contracts are defined.
- Automatic replay of uncertain side-effecting nodes after crashes.
- Performance-oriented handler/version rollout mechanisms.
- Persistent device registry, aliases, provider correlation, and emulator
  lifecycle management.

Handler versioning is deferred, but these versions remain necessary:

- Runtime application version.
- Runtime IR format version.
- `minimumRuntimeVersion` compatibility checks.

## 3. Proposed logical architecture

These are logical module boundaries, not yet approved folder names:

```text
Runtime Agent
├── Contracts
│   ├── Runtime IR
│   ├── Local API
│   ├── Session device selection rules
│   ├── Execution state
│   ├── Events and errors
│   └── Typed device commands
├── Agent foundation
│   ├── Configuration
│   ├── Supervisor
│   ├── Structured logging
│   └── Local authentication
├── Validation subsystem
│   ├── Editor Graph validator
│   ├── Deterministic diagnostics
│   ├── Supported-node contract checks
│   └── Runtime IR admission validator
├── Device subsystem
│   ├── Safe process runner
│   ├── ADB toolchain
│   ├── Generic ADB discovery
│   ├── Readiness probe
│   └── Session inventory
├── Execution subsystem
│   ├── IR admission gate
│   ├── Binding resolver
│   ├── Handler registry
│   ├── Workflow transition engine
│   ├── Device execution runner
│   └── Multi-device scheduler
├── Persistence
│   ├── SQLite store
│   ├── Atomic checkpoints
│   ├── Execution events
│   └── Artifact metadata
├── Integrations
│   ├── Studio Development mode
│   └── Launcher Production mode
└── Node handlers
    ├── Flow handlers
    └── Android handlers
```

The Go design should use explicit constructors, small interfaces defined by their consumers, context propagation, bounded concurrency, and a clear owner for every goroutine.

## 4. Detailed implementation sequence

Each numbered item is one independently reviewable checkout. We stop for review after each one.

### Phase A — Freeze contracts before writing the engine

| ID  | Module                     | Deliverable                                                                                                                                  | Checkout condition                                                                      |
| --- | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| A1  | Contract decision register | Decide Runtime IR version representation, local transport, event transport, ADB resolution policy, artifact location, and identifier formats | No unresolved decision blocks schema authoring                                          |
| A2  | Runtime IR v1 contract     | Strict workflow, node, edge, binding, timeout, retry, error-policy, capability, and metadata rules                                           | All 11 current nodes can be represented; unknown fields and invalid graphs are rejected |
| A3  | Session device selection   | Freeze ephemeral ADB discovery, selection by current serial, start-time revalidation, and no-Device-entity rule                              | Device ownership does not overlap A5, A6, or A7                                         |
| A4  | Typed device command contract | Seven closed Android commands; non-screenshot success has no payload and Screenshot returns only `artifactId`                              | Workflow data cannot control an executable, raw arguments, target serial, shell syntax, or artifact path |
| A5  | Execution schemas          | Three persisted records, reduced lifecycle states, atomic checkpoint operation, single-active-run invariant, and derived summary             | Every transition, selected serial, node output, and interruption has one owner          |
| A6  | Event and error schemas    | Four typed state events, Runtime-wide replay cursor, reduced stable errors, retryability, expiry recovery, and sanitization                   | Client can replay changes or recover from a watermarked snapshot without parsing logs   |
| A7  | Local API schema           | Protected token handoff, compatibility info, validation, devices, idempotent start, status/control, watermarked snapshot, SSE replay, and screenshots | Every endpoint has strict request, response, error, authentication, and cursor semantics |
| A8  | Contract fixtures          | Versioned manifest plus 19 hashed Editor Graph, Runtime IR, event/error, Local API, and SSE cases                                             | Canonical bytes and expectations are reviewable; later Go and TypeScript suites consume them |

A1 through A8 are accepted. Their authoritative decisions are recorded in
[`phase_a.md`](./phase_a.md).

Important contract rules:

- Unknown fields are rejected at trust boundaries.
- Collection sizes and string lengths are bounded.
- `formatVersion` versions the Runtime IR structure.
- Registry lookup is `runtimeHandler → handler`; there is no handler-version key yet.
- The Runtime validates node inputs and outputs against the approved node contracts.
- Node Definition changes cannot silently change the Runtime contract.
- A3 owns only selection rules; A5 owns persisted selected serials, A6 owns
  failures, and A7 owns device-related JSON payloads.
- A6 events describe committed A5 changes; they do not replace the A5 state
  snapshot or B4 operational logs.
- A7 exposes no generic CRUD: API projections cannot mutate A5 records or
  extend A6 payloads.

### Phase B — Runtime process foundation

| ID  | Module                | Depends on | Deliverable and checkout condition                                                                                  |
| --- | --------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------- |
| B1  | Go project skeleton   | A1         | Minimal executable, internal packages, build metadata, and test layout; no unnecessary framework                    |
| B2  | Configuration         | B1         | Validated configuration for mode, loopback address, ADB path, database, artifacts, ordinary HTTP/SSE timeouts, and concurrency limits |
| B3  | Supervisor            | B1–B2      | Root context, component startup ordering, graceful shutdown, and bounded shutdown timeout                           |
| B4  | Logging and redaction | B1         | Structured logs containing run/device/node correlations without secrets or protected workflow payloads              |
| B5  | Local authentication  | A7, B2     | Per-startup random token, protected per-user connection-file handoff, constant-time verification, and cleanup       |
| B6  | Minimal local API     | B3–B5      | Loopback-only minimal health plus the authenticated A7 runtime-information handshake                                |

Gate: the agent starts, reports readiness, rejects unauthenticated requests, and exits cleanly without goroutine leaks.

### Phase C — Local workflow validation

This phase is an early Runtime reliability boundary. It gives Studio local,
deterministic authoring feedback before compilation and before any persistence,
ADB discovery, run creation, or device command. It follows the approved Backend
Editor Graph behavior but is implemented independently in Go.

| ID | Module | Deliverable and checkout condition |
|---|---|---|
| C1 | Validation contracts | Implement the A7 result envelope, stable diagnostic codes, deterministic ordering, diagnostic cap, and payload limits |
| C2 | Editor Graph decoder | Strictly decode the supported Editor Graph version and reject unknown fields and invalid bounded values |
| C3 | Topology validator | Validate limits, unique IDs, Start/entry, endpoints, End rules, reachability, and acyclicity using bounded traversals |
| C4 | Node-contract validator | Validate node types, source ports, and static configuration against the Runtime's immutable supported-node registry |
| C5 | Local validation API | Expose authenticated content-only validation for Studio without saving, compiling, enqueuing, or executing the workflow |
| C6 | Cross-language parity | Vendor and run A8 Editor Graph fixtures against Backend/TypeScript and Runtime/Go; compare classifications and ordered diagnostic codes |

The local validator answers whether an Editor Graph is valid for authoring. It
does not authorize execution and does not validate device availability. A saved
`validationSummary` is evidence of an earlier check, not a trusted Runtime
decision.

Gate: Studio can submit valid and invalid Editor Graph fixtures and receive the
same bounded, deterministic result from repeated calls. Validation performs no
database write, creates no run, and invokes no ADB command.

### Phase D — Persistence and observability foundation

Persistence comes before workflow execution so the first real execution is observable.

| ID  | Module                       | Deliverable and checkout condition                                                                  |
| --- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| D1  | SQLite schema and migrations | Reviewed tables, unique RunGroup request IDs, constraints, indexes, migration mechanism, and retention model       |
| D2  | Execution repository         | Context-aware, parameterized state reads/writes with explicit transaction boundaries                |
| D3  | Event repository             | A1 Runtime-wide monotonic cursor allocation, retention, and reconnect queries                        |
| D4  | Atomic checkpoint writer     | Attempt result, validated node output, next position, execution state, and resulting events persisted in one transaction |
| D5  | Artifact store               | Binary files outside SQLite; safe paths, hashes, dimensions, media type, and metadata inside SQLite |
| D6  | Startup recovery             | Previously active executions become `INTERRUPTED`; no automatic uncertain replay                    |

Gate: committed state always exists before an event is broadcast.

### Phase E — Generic Android device subsystem

| ID  | Module                 | Deliverable and checkout condition                                                                                                  |
| --- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| E1  | Safe process runner    | Executes an explicit binary with separate arguments, timeout, cancellation, bounded output, exit code, stdout, stderr, and duration |
| E2  | ADB toolchain resolver | Selects a Runtime-controlled or explicitly configured ADB executable                                                               |
| E3  | ADB discovery parser   | Parses `adb devices -l`, including device, offline, unauthorized, and malformed output                                              |
| E4  | Device readiness probe | Checks boot completion and fixed command prerequisites without collecting unused platform or screen metadata                        |
| E5  | Session inventory      | Keeps the latest discovery result in memory only; no Device table, UUID, alias, history, or provider correlation                    |
| E6  | Selection preflight    | Rejects duplicate serials and revalidates every selected endpoint immediately before run admission                                  |
| E7  | Device API             | Implements the A7 discovery and refresh responses without creating persistent device records                                        |

Gate:

- A physical Android device works using only ADB.
- Android Studio Emulator and other ADB-visible emulators use the same
  session endpoint path.
- Restarting Runtime discards the inventory without losing execution history,
  because A5 persists the accepted serial on each device execution.

### Phase F — Real device commands and current handlers

| ID  | Module             | Real operation                                                                                              |
| --- | ------------------ | ----------------------------------------------------------------------------------------------------------- |
| F1  | Command dispatcher | Binds every command to one explicit ADB serial and rejects arbitrary commands                               |
| F2  | Launch App         | Dispatches launch for the configured package; success has no workflow-visible payload                       |
| F3  | Tap                | Performs an absolute-coordinate tap                                                                         |
| F4  | Swipe              | Performs an absolute-coordinate swipe with bounded duration                                                 |
| F5  | Input Text         | Safely encodes the configured text without shell interpretation                                             |
| F6  | Key events         | Implements Back and Home                                                                                    |
| F7  | Screenshot         | Captures a real PNG, verifies it, extracts dimensions, and stores it as an artifact                         |
| F8  | Flow handlers      | Implements Start, End, and cancellable Delay; implements Log only after its sensitive-data behavior is approved |
| F9  | Handler registry   | Registers exactly one handler for every supported `runtimeHandler`; rejects duplicates and unknown handlers |

A fake device executor may exist only as a test double. Passing a fake test does not satisfy the product checkout. Each Android handler needs a real-device acceptance test.

Command completion must distinguish:

- Command dispatched successfully.
- Device operation observed successfully.
- Business result confirmed by a later explicit observation.

For example, a successful tap means ADB accepted the tap command; it does not prove that a purchase, login, or other business outcome occurred.

### Phase G — Workflow engine

| ID  | Module                   | Deliverable and checkout condition                                                                                             |
| --- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| G1  | Runtime IR admission validator | Strictly validates the exact executable document, version, graph, bindings, workflow inputs, handler availability, compatibility, and size limits before run creation |
| G2  | Binding resolver               | Resolves literals, per-device workflow inputs, and prior node outputs without cross-device leakage                                                |
| G3  | Transition engine              | Supports deterministic success and error edges; no condition, switch, or loop semantics yet                                                        |
| G4  | Timeout and retry engine        | Per-attempt timeout, bounded retries, recorded attempts, cancellation, and retry classification                                                    |
| G5  | Single-device runner            | Executes nodes sequentially for one device and persists every transition                                                                           |
| G6  | Output validator                | Rejects handler results that violate the declared node output schema                                                                               |
| G7  | Final result calculator         | Produces authoritative device-execution status and summary from persisted results                                                                  |

`G1` is mandatory even when Phase C, Studio, or Backend previously reported the
workflow as valid. It validates the exact Runtime IR and execution inputs at the
trust boundary. Failure creates no active run and dispatches no device command.

Gate: one real device can run a complete workflow using all applicable current nodes, including an error branch and retry/timeout behavior.

### Phase H — Multi-device scheduling and lifecycle

| ID  | Module                   | Deliverable and checkout condition                                                          |
| --- | ------------------------ | ------------------------------------------------------------------------------------------- |
| H1  | Run-group coordinator    | After G1 and device preflight pass, idempotently admits at most one non-terminal run group by A7 request ID and creates one persisted device execution per accepted serial |
| H2  | Bounded scheduler        | Starts only the configured number of device executions concurrently and queues the rest     |
| H3  | Per-device worker        | One supervised goroutine per active device execution; nodes remain sequential within it     |
| H4  | Failure isolation        | One device failure does not stop unaffected devices under the fixed MVP rule                 |
| H5  | Pause/resume             | Stops before the next node, persists a safe checkpoint, and resumes only from a valid state |
| H6  | Cancellation             | Propagates context cancellation into delays and ADB processes before releasing run resources |
| H7  | Device disconnect policy | Fails only the affected device execution with the approved A6 device error                  |
| H8  | Group result aggregation | Derives group counters and terminal state from persisted device-execution states            |

Context ownership should be:

```text
Agent context
└── Run-group context
    └── Device-execution context
        └── Node-attempt timeout context
```

Gate: two or more real devices can execute independently, concurrency is bounded, data is isolated, and all resources are released at termination.

### Phase I — Client integration

#### I1. Studio Development mode

- Validate the current Editor Graph locally and render deterministic diagnostics.
- Studio sends strict draft Runtime IR directly.
- List and select ready ADB endpoints.
- Run the complete draft workflow.
- Stream typed execution events.
- Reconnect using the last persisted Runtime-wide event cursor.
- Query the authoritative snapshot if events were missed.
- Inspect screenshots, node attempts, failures, and final summaries.

This does not depend on Submission, Product, or Product Release.

Test-one-node and arbitrary run-from-node modes are deferred until their input,
`NODE_OUTPUT` dependency, stopping, and result contracts are explicitly
defined. They do not change the A5 full-workflow execution state.

Publisher-authored `utility.log` persistence or streaming is also deferred
until its workflow-input and sensitive-data rules are approved. Operational
Runtime logs remain B4 scope and are not A6 execution events.

#### I2. Launcher Production mode

This waits for the Product–Submission–Release backend group.

It includes:

- Installed package format.
- Release identity and integrity verification.
- Runtime compatibility verification.
- Customer authorization or lease validation.
- Capability and device-limit enforcement.
- Protected Runtime IR loading.
- No plaintext workflow persistence.
- Generic `PAUSED` or a policy-defined terminal result when authorization expires.
- Launcher handshake and production execution APIs.

The execution engine is reused; Production mode adds a protected workflow source and authorization boundary.

### Phase J — MVP hardening and release gate

Before considering the MVP releasable:

- Unit tests for parsers, validators, transitions, state machines, and result calculation.
- Hash-verified A8 Editor Graph, Runtime IR, event/error, Local API, and SSE
  contract tests in every applicable Go and TypeScript consumer.
- Tests proving validation failure creates no run, database execution record,
  ADB process, or partial multi-device execution.
- Integration tests using controlled ADB/process test doubles.
- SQLite transaction and restart-recovery tests.
- HTTP authentication and strict-decoding tests.
- Start-request idempotency tests, including a lost-response retry.
- SSE replay/live handoff, slow-client disconnect, and expired-cursor recovery tests.
- Race detection.
- Goroutine leak checks.
- Fuzzing for ADB output and Runtime IR decoding.
- Command-injection and path-traversal tests.
- Real-device matrix:
  - Physical Android device.
  - Official Android Emulator on Ubuntu.
- Single-device end-to-end test.
- Multi-device failure-isolation test.
- Client disconnect/reconnect test.
- Agent crash and `INTERRUPTED` recovery test.
- Log redaction and artifact-access review.

Tests remain part of each proposed checkout, but I will confirm the test scope with you before implementing that checkout, as requested.

## 4. Release milestones

| Milestone                   | Observable result                                                |
| --------------------------- | ---------------------------------------------------------------- |
| M1 — Contract ready         | Contracts and cross-language fixtures approved                   |
| M2 — Validation ready       | Studio receives bounded local diagnostics without execution      |
| M3 — Device ready           | Runtime lists, selects, and revalidates generic ADB endpoints     |
| M4 — Command ready          | All seven Android operations execute on real devices             |
| M5 — Engine ready           | All 11 current handlers run on one device with persisted events  |
| M6 — Multi-device ready     | Bounded, isolated concurrent execution works                     |
| M7 — Studio MVP             | Draft workflows can be tested end-to-end from Studio             |
| M8 — Production MVP         | Launcher executes authorized installed releases                  |
| M9 — Post-release evolution | Handler versioning, advanced nodes, and advanced graph semantics |

## 5. Existing documents that conflict with this plan

When you approve documentation work, these areas need revision:

- The current discovery plan is too dependent on LDPlayer’s bundled ADB.
- The current mapping plan prefers LDPlayer-specific control.
- Older exploratory documents still describe node-handler versions and
  transports that Phase A has superseded.
- They include conditional/loop behavior and connector execution that are now deferred.
- The existing draft starts persistence/events too late; they must precede real workflow execution.
- Older plans do not separate local Editor Graph diagnostics from mandatory
  Runtime IR execution admission.

Relevant current documents include:

- [Runtime functional requirements](/home/dino/dev/a2matex/a2matex-plan/specifications/3.1.4_3.2.5_a2matex_runtime_functional_requirements.md)
- [Runtime understanding](/home/dino/dev/a2matex/a2matex-plan/components/a2matex-runtime/understanding.md)
- [Existing draft implementation plan](/home/dino/dev/a2matex/a2matex-plan/components/a2matex-runtime/draft_implementation_plan.md)
- [LDPlayer and ADB discovery plan](/home/dino/dev/a2matex/a2matex-plan/components/a2matex-runtime/4_detect_ldplayer_instances_and_adb.md)
- [Current Backend Android node definitions](/home/dino/dev/a2matex/a2matex-be/src/routes/node-catalog/definitions/android-node-definitions.ts)
- [Current Backend flow node definitions](/home/dino/dev/a2matex/a2matex-be/src/routes/node-catalog/definitions/flow-node-definitions.ts)

The Go guidance materially influenced the bounded concurrency hierarchy, explicit context ownership, small consumer-owned interfaces, transaction-first event persistence, and verification gates.
