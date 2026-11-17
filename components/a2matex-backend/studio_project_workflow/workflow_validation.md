# Cloud Workflow Validation Feature Spec

**Status:** Backend core complete; node execution-setting extension pending implementation

**Phase:** `a2matex-be/phase4.md` / Studio workflow checklist phase 7

## 1. Purpose

Validate Editor Graph content and return deterministic diagnostics. The same service supports:

1. a temporary authenticated Backend testing endpoint; and
2. a future background worker processing immutable submitted workflow content.

Studio editing validation runs locally through the Go Runtime Agent. Cloud save does not require a
valid workflow.

## 2. Scope

Included:

- Editor Graph schema version 1 contract/type validation;
- graph topology;
- active Node Catalog type and source-port checks;
- static node `config` validation;
- workload limits and deterministic diagnostics.

Accepted by the next compiler slice but not implemented by the current validator:

- optional node-level timeout and retry settings;
- Runtime IR consistency and compilation.

Deferred:

- binding values and references;
- controlled-loop semantics and future expression forms;
- assets and credentials;
- result persistence, queue orchestration, encryption, packaging, and execution.

## 3. Service contract

```ts
validate(editorGraph: unknown): Promise<WorkflowValidationOutput>
```

Pipeline:

1. enforce the workflow byte limit;
2. parse the strict graph contract internally;
3. return contract/type problems as diagnostics;
4. run topology checks;
5. load distinct active Node Definitions in one batched query;
6. run catalog, source-port, and config checks; and
7. sort/cap diagnostics and derive status.

## 4. Temporary HTTP adapter

```http
POST /api/v1/workflow-validation
Authorization: Bearer <access-token>
Content-Type: application/json
```

```json
{
  "editorGraph": {}
}
```

The outer request contains exactly one JSON-object `editorGraph`. Missing/invalid outer fields
return `400`/`422`. `@SkipPermissionCheck()` skips route RBAC only; the global guard still requires
a valid token, active session, active user, active role, and active device.

## 5. Result semantics

```json
{
  "status": "INVALID",
  "errors": [
    {
      "code": "EDGE_SOURCE_PORT_NOT_FOUND",
      "message": "Edge source port does not exist on the source node definition",
      "nodeId": "start_01",
      "edgeId": "edge_01",
      "path": ["edges", 0, "sourcePort"]
    }
  ],
  "warnings": [],
  "truncated": false
}
```

- `VALID`: no diagnostics.
- `VALID_WITH_WARNINGS`: warnings and no errors.
- `INVALID`: contract, topology, catalog, port, or config errors.
- HTTP `200`: validation completed, including `INVALID`.
- HTTP `400`/`422`: malformed outer request.
- HTTP `401`: authentication/session failure.
- HTTP `413 Error.EditorGraphPayloadTooLarge`: graph exceeds 1 MiB.
- HTTP `500`: unexpected catalog, database, or server failure.

## 6. Graph contract and schema version

**Implemented in Backend and local Runtime validation:** The strict node `editor`
object accepts `label` and `description`, each a non-empty string or `null`.
Reject empty strings and other types. Accept omission in existing graphs and
handle it as `null`. Existing `x` and `y` remain
required. Studio falls back to the definition name for a null label and
hides a null note. The fields are saved with the project graph,
have no semantic validation effect, and are omitted from Runtime IR.
`editor.x` and `editor.y` are Studio canvas coordinates. The local Runtime
validator receives them because it validates the full Editor Graph.
Android targeting uses the action's target mode and input bindings.

Editor Graph v1 requires:

- positive integer `schemaVersion` with supported value `1`;
- bounded identifiers and valid node type names;
- `entryNodeId`, node array, and edge array;
- JSON-object `config` and `bindings`;
- optional `timeoutMs` integer from 1 to 600,000;
- optional strict `retryPolicy` with optional `maxAttempts` from 1 to 3 and optional `delayMs` from
  0 to 60,000;
- `STOP` or `ERROR_BRANCH` policy;
- finite editor coordinates;
- strict graph/node/edge fields.

Contract/type failures use `EDITOR_GRAPH_CONTRACT_INVALID`. Another positive schema version uses
`GRAPH_SCHEMA_VERSION_UNSUPPORTED`.

Compilation resolves missing execution settings without mutating the Editor Graph:

```text
timeoutMs = node.timeoutMs ?? nodeDefinition.defaultTimeoutMs
maxAttempts = node.retryPolicy.maxAttempts ?? 1
delayMs = node.retryPolicy.delayMs ?? 0
```

The Runtime IR node always contains all three effective values. For
`flow.wait_variable`, `config` contains only the Global key `name`; the effective
node-level `timeoutMs` is the wait deadline.

`schemaVersion` versions the graph JSON structure, not the Node Catalog. Adding node types keeps
existing graphs on version `1`. A future breaking graph-format version should have its own parser
and normalize to the shared internal representation before semantic validation.

### Planned targeting in Editor Graph v1

**Stage 1:** Backend and local Runtime validate explicit targeting modes.
Studio integration, submission gating, and executable targeting have separate
delivery gates.

Keep `schemaVersion: 1` and the existing node envelope. The supported authoring
`config.targetMode` values for Tap, Swipe, and Long Press are
`ABSOLUTE_PIXELS`, `SCREEN_PERCENT`, and `SELECTOR_ID`, where supported by the
node. Tap, Swipe, and Long Press all require an explicit mode. Pixel drafts
must specify `ABSOLUTE_PIXELS` and supply the required pixel literal bindings.
Omitting the mode produces `NODE_CONFIG_INVALID` at `config.targetMode`.
Put the mode's input bindings in the existing `bindings` object. For example,
a selector Tap uses `config: { "targetMode": "SELECTOR_ID" }` and a
`bindings.selector_id: { "kind": "LITERAL", "value": "com.example.app:id/login" }`.
Do not save the
UI hierarchy or dummy pixel values.

Validate exactly one allowed mode per node against its current
`editorHints.targeting.modes` definition from the active Node Catalog. The
[catalog targeting matrix](node_catalog.md#targeting-modes-and-long-press)
summarizes those schemas.
Require all input bindings for that mode and reject target fields from other
modes. A Swipe uses one mode for both endpoints. Keep the action's duration
binding where required. Explicit modes require strict literal objects.
Invalid mode configuration yields `NODE_CONFIG_INVALID`. Invalid target
bindings yield `NODE_BINDINGS_INVALID` with the binding path.

The revised Screenshot definition requires its own explicit `targetMode`.
`FULL_SCREEN` takes zero bindings. `ABSOLUTE_PIXELS` and `SCREEN_PERCENT`
each require the four literals for their own coordinate unit. Validate the
leading edge is strictly before the trailing edge on each axis when the
fields are individually valid. Report bad ordering at the trailing edge's
`value` with `NODE_BINDINGS_INVALID`. F7 checks absolute edges against the
captured frame during execution. See the
[F7 mode contract](../../a2matex-runtime/f_device_commands/f7_screenshot.md#1-node-contract).

Every node accepts `STOP`. `ERROR_BRANCH` is valid only when the active Node
Definition declares the static `error` output-port key. Otherwise Backend and
Runtime return `NODE_ERROR_POLICY_INVALID` at the node's `errorPolicy` path.

Authoring validation can accept these drafts before Runtime can execute them.
Studio must identify unsupported execution modes, invalidate any stale
compiled IR after a targeting change, and prevent test runs and submission
until supported. Backend submission must independently reject planned nodes
and modes; saving a draft is not approval to execute it. No new graph or
Runtime IR version is introduced by this update.

## 7. Semantic rules

Stage 3 adds direct output and Flow variable binding checks and Set Variable
authoring validation. The existing batched catalog lookup also loads input
and output schemas. See the
[shared binding contract](../../a2matex-runtime/contracts/workflow_bindings.md).

- Unique node and edge IDs.
- Exactly one Start matching `entryNodeId`.
- Existing edge endpoints and source ports.
- No outgoing edge from End.
- Every node reachable from Start.
- At least one reachable End.
- No directed cycles in the MVP.
- Every node type active in the current catalog.
- Static config required/type/range/pattern/additional-property checks.

## 8. Limits, access, and safety

- Maximum serialized Editor Graph: 1 MiB.
- Maximum nodes: 500.
- Maximum edges: 1,000.
- Maximum returned diagnostics: 100 with deterministic sorting and `truncated` flag.
- One Node Definition query regardless of repeated node instances.
- Validators perform no database operations.
- No migration, write, transaction, lock, cache, or queue is required.
- No feature success logs or graph/config/binding payload logging.

## 9. Implementation map

| File                                  | Responsibility                            |
| ------------------------------------- | ----------------------------------------- |
| `workflow-validation.controller.ts`   | Authenticated testing adapter             |
| `workflow-validation.service.ts`      | Reusable validation pipeline              |
| `workflow-validation.repo.ts`         | Batched active Node Definition query      |
| `validator/editor-graph.validator.ts` | Topology, catalog checks, result assembly |
| `validator/node-config.validator.ts`  | MVP config-schema interpreter             |
| `workflow-validation.model.ts`        | Request, graph, and response contracts    |
| `workflow-validation.constant.ts`     | Limits, codes, statuses, safe messages    |
| `guidance/README.md`                  | Module behavior and invariants            |
| `guidance/HOW_TO_READ.md`             | Code-reading path                         |
| `guidance/MANUAL_TESTING.md`          | Postman payloads and expected results     |

## 10. Verification

- [x] Contract, topology, catalog, config, repository, service, and controller tests pass.
- [x] Full Backend suite passes: 18 files, 79 tests.
- [x] TypeScript build passes.
- [x] Type-aware lint completes with only unrelated existing warnings.
- [x] `git diff --check` passes.
- [x] Manual Postman cases documented for tester execution.

Manual test guide:
[`a2matex-be/src/routes/workflow-validation/guidance/MANUAL_TESTING.md`](../../../../a2matex-be/src/routes/workflow-validation/guidance/MANUAL_TESTING.md).
