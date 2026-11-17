# Node Catalog Feature Spec

**Status:** Implemented and verified on 2026-10-09  
**Consumers:** Studio and Runtime integration  
**Backend source:** `a2matex-be/src/routes/node-catalog`

## 1. Purpose and boundary

Backend is the source of truth for system-predefined workflow nodes. Studio fetches the current
catalog to build its palette and inspectors. Runtime uses the same node `type` and handler contract.

Included: definition validation, PostgreSQL persistence, system publication, Runtime compatibility
filtering, protected read API, catalog hashing and HTTP cache validation.

Deferred: user-defined nodes, node-level versions, immutable historical definition lookup,
deprecation state, arbitrary code, Ajv/standard JSON Schema, conditional/loop nodes, retry policy,
parameter binding and executable Runtime handler verification.

## 2. Final MVP decisions

- `NodeDefinition.type` is the unique stable identity; one mutable current row exists per type.
- Status is `ACTIVE` or `DISABLED`. Removed source definitions are disabled, not deleted.
- Definition JSON uses a strict Backend-owned schema envelope. It has no `$schema` or nested schema
  version field.
- Source definitions are system-owned and cannot be published through a public API.
- `minimumRuntimeVersion` and the request version are strict `X.Y.Z`; a node is returned when the
  caller version is greater than or equal to its minimum version.
- `contentHash` is SHA-256 of one definition. `catalogHash` is SHA-256 of the ordered compatible
  response nodes and is used as the ETag.
- `STOP` is always supported. `ERROR_BRANCH` is supported only when the
  definition contains the static `error` output port.

## 3. Persisted model

### `Capability`

Unique `code`, display metadata, `isSensitive`, timestamps and soft-delete field. Current codes:

- `android.device.control`
- `android.screen.capture`

### `NodeDefinition`

Required fields:

| Group            | Fields                                                        |
| ---------------- | ------------------------------------------------------------- |
| Identity/display | `id`, `type`, `name`, `description`, `category`, `status`     |
| Runtime          | `runtimeHandler`, `minimumRuntimeVersion`, `defaultTimeoutMs` |
| Contract         | `configSchema`, `inputSchema`, `outputSchema`, `outputPorts`  |
| Studio           | `editorHints`                                                 |
| Sync/audit       | `contentHash`, `createdAt`, `updatedAt`, `deletedAt`          |

Database invariants: unique `type`, positive timeout, 64-character lowercase hex hash, all five JSON
fields are objects, and index `(status, type)`.

### `NodeDefinitionCapability`

Joins a current definition to a capability. The pair `(nodeDefinitionId, capabilityId)` is unique.
Foreign keys use the database default `NO ACTION` behavior.

## 4. Definition contract

Each definition declares:

- identity/display: `type`, `name`, `description`, `category`;
- execution: `runtimeHandler`, strict `minimumRuntimeVersion`, and positive default timeout used
  when a node instance omits `timeoutMs`;
- three object schemas with `type: "object"`, `properties`, `required`, and
  `additionalProperties: false`;
- unique static output-port keys; `flow.switch` additionally resolves its ordered rule IDs and
  `default` as instance ports from validated config;
- allowed binding sources and their saved fields in each self-describing `inputSchema` property;
- optional targeting-mode binding schemas in `editorHints.targeting`;
- capability codes.

Required fields must exist in `properties`; duplicate ports and capabilities are
invalid. Port labels, port kinds, icons, documentation, field ordering, and
target-mode labels belong to Studio. `ERROR_BRANCH` is derived from the presence
of the `error` port rather than duplicated in each definition. No executable
expression or JavaScript is accepted.

### Reduced Studio metadata

Studio maps icon, color, field layout, port labels, and target-mode labels from stable identifiers, with fallbacks for unknown values. The Backend keeps only semantic targeting and binding schemas required to avoid duplicating business
options in Studio. Catalog hashing and ETag continue
to cover the returned catalog content.

## 5. Initial catalog

| Category | Node types                                                                                                                                                         |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Flow     | `flow.start`, `flow.end`, `flow.delay`, `flow.wait_variable`, `flow.switch`, `flow.set_variable`                                                                   |
| Utility  | `utility.log`                                                                                                                                                      |
| Android  | `android.launch_app`, `android.tap`, `android.swipe`, `android.input_text`, `android.press_back`, `android.press_home`, `android.screenshot`, `android.long_press` |

The Backend source catalog contains 15 definitions.

Special Flow contracts:

- `flow.set_variable` requires `config.name` and explicit `scope` (`FLOW` or `RUN_GROUP`), and
  receives its value through `bindings.value`;
- `flow.wait_variable` requires only `config.name`; the node instance's optional `timeoutMs`, or the
  definition default when omitted, is the wait deadline;
- `flow.switch` owns ordered `config.rules`; each rule ID and `default` is a required instance output
  port, with conditional `error` when the node uses `ERROR_BRANCH`.

### Set Variable authoring contract

`flow.set_variable` stores one resolved scalar under a case-sensitive name. `FLOW` belongs to one
device execution. `RUN_GROUP` is read through `GLOBAL_VARIABLE` and belongs to the current Runtime
Run Group. Studio initializes scope to `FLOW` from the schema default and always saves it explicitly;
Backend rejects a missing or unsupported scope. The required `inputSchema.value` property publishes
the allowed sources and the fields saved for each source:

```json
{
  "type": "flow.set_variable",
  "configSchema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "pattern": "^[A-Za-z_][A-Za-z0-9_]{0,99}$"
      },
      "scope": {
        "type": "string",
        "enum": ["FLOW", "RUN_GROUP"],
        "default": "FLOW"
      }
    },
    "required": ["name", "scope"],
    "additionalProperties": false
  },
  "inputSchema": {
    "type": "object",
    "properties": {
      "value": {
        "bindingKinds": {
          "LITERAL": {
            "fields": {
              "value": { "types": ["string", "number", "boolean"] }
            }
          },
          "WORKFLOW_INPUT": { "fields": ["inputName"] },
          "NODE_OUTPUT": { "fields": ["nodeId", "field"] },
          "FLOW_VARIABLE": { "fields": ["name"] },
          "GLOBAL_VARIABLE": { "fields": ["name"] }
        }
      }
    },
    "required": ["value"],
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {},
    "required": [],
    "additionalProperties": false
  },
  "outputPorts": { "ports": ["success", "error"] },
  "editorHints": {}
}
```

Studio renders `configSchema` into `node.config` and each `inputSchema` property into the matching
`node.bindings` key. For `value`, the selected binding kind declares exactly which fields Studio
collects and saves. The schemas are never copied into a Project Draft. Backend authoring validation
requires `bindings.value`, rejects unpublished binding kinds and additional binding keys, and accepts
only string, finite-number, or boolean literal values.

Catalog publication does not claim Runtime execution support. The variable store, binding resolver,
scope-aware Set Variable handler, and atomic checkpoint write remain owned by Runtime Phase G/H.

### Switch authoring contract

`flow.switch` evaluates one shared `bindings.value` against one to 32 ordered rules and follows the
first matching branch. Studio generates each rule ID as a UUID, never exposes it as an editable
field, and preserves it during edits and reordering. The UUID joins three pieces of instance data:

```text
config.rules[].id = bindings[rule.id] key = edge.sourcePort
```

The definition publishes:

- `configSchema.rules`: ordered UUID/operator records;
- operator options with supported value types and `operandCount`;
- `inputSchema.properties.value`: the shared left-side binding contract;
- `inputSchema.dynamicInputs.ruleOperand`: the per-rule operand contract, keyed by rule UUID;
- `outputPorts.dynamicPorts`: rule UUIDs as branch ports, plus static `default` and conditional
  `error` ports.

Binary operators `EQ`, `NE`, `LT`, `LTE`, `GT`, `GTE`, `CONTAINS`, `NOT_CONTAINS`, `STARTS_WITH`,
and `ENDS_WITH` have `operandCount: 1` and require `bindings[rule.id]`. Unary operators `IS_EMPTY`,
`IS_NOT_EMPTY`, `IS_TRUE`, and `IS_FALSE` have `operandCount: 0` and forbid that binding. Both the
shared value and binary operands accept `LITERAL`, `WORKFLOW_INPUT`, `NODE_OUTPUT`, `FLOW_VARIABLE`,
and `GLOBAL_VARIABLE` using the same field contracts as Set Variable.

Static type rules are:

| Operators                                              | Required resolved type                           |
| ------------------------------------------------------ | ------------------------------------------------ |
| `EQ`, `NE`                                             | matching string, number, or boolean values                |
| `LT`, `LTE`, `GT`, `GTE`                               | number on both sides; Runtime compares floating-point values |
| `CONTAINS`, `NOT_CONTAINS`, `STARTS_WITH`, `ENDS_WITH` | string on both sides                             |
| `IS_EMPTY`, `IS_NOT_EMPTY`                             | string left value                                |
| `IS_TRUE`, `IS_FALSE`                                  | boolean left value                               |

Unknown variable types are checked when Runtime resolves them. Every rule and `default` requires one
outgoing edge. `error` requires one edge only when the node selects `ERROR_BRANCH`. Switch has no
bindable output data fields. Range operators, regex, null, object keys, nested conditions, and
fall-through are outside the MVP contract.

### Targeting modes and Long Press

The seed includes Long Press. Catalog presence does not indicate Runtime handler
availability. Tap, Swipe, and Long Press publish their form contract through
`editorHints.targeting.modes`. Each mode contains `mode` and a strict
`bindingSchema`. Their top-level `inputSchema` is empty because
the selected mode owns the valid bindings. Backend and local Runtime validate
the explicit authoring modes below. Executable support belongs to Stage 2.

| Node type                  | Absolute pixels (`ABSOLUTE_PIXELS`)           | Screen percentage (`SCREEN_PERCENT`)                           | Element ID (`SELECTOR_ID`) | Additional input                                                                 |
| -------------------------- | --------------------------------------------- | -------------------------------------------------------------- | -------------------------- | -------------------------------------------------------------------------------- |
| `android.tap`              | `x`, `y` (existing)                           | `xPercent`, `yPercent`                                         | `selector_id`              | None                                                                             |
| `android.swipe`            | `startX`, `startY`, `endX`, `endY` (existing) | `startXPercent`, `startYPercent`, `endXPercent`, `endYPercent` | Unsupported                | Existing gesture `durationMs` in both modes                                      |
| `android.long_press` (new) | `x`, `y`                                      | `xPercent`, `yPercent`                                         | `selector_id`              | Required positive safe integer hold `durationMs`. Stage 2 plans a 60,000 ms cap. |

- Percentages are integers from 0 to 100, measured from the top-left of the
  current device screen in its current orientation. Resolve each axis against
  its own dimension: `round(percent × (dimension - 1) / 100)`. Scaling does
  not compensate for controls that move after a layout or orientation change.
- `selector_id` is the non-empty, full Android UI XML `resource-id`, for
  example `com.example.app:id/login_button`. At execution, resolve an exact
  match in the current hierarchy and act at the center of its visible bounds.
  Missing matches, multiple matches, or unusable bounds fail the action;
  never fall back to coordinates. The authoring XML is not an execution input.
- Long Press is an Android device-control action. The Runtime targeting
  adapter and workflow handler are planned for Phase F.
  the existing Input Text remains a focused-field action and can follow Tap.
  No input changes for the other nine existing nodes. Scroll is out of scope.
- The [Editor Graph rules](workflow_validation.md#planned-targeting-in-editor-graph-v1)
  describe the validated draft representation. Runtime IR remains unchanged.

Studio must render the target-mode selector from `editorHints.targeting.modes`
and render only the selected mode's `bindingSchema`. The Project Draft saves the
selected string in `config.targetMode` and only that mode's strict literal
bindings. When the user changes mode, Studio removes the previous mode's target
bindings before saving; it never stores every mode's fields on one node.

## 6. Publication

Run the explicit backend command `npm run seed:node-catalog`. Despite the script name, it is an
idempotent catalog synchronizer and does not modify roles or permissions.

Publication must:

1. validate all definitions and capability references before opening a transaction;
2. upsert capabilities and definitions in set-based operations;
3. replace current capability links for the published definitions;
4. set absent definitions to `DISABLED`;
5. commit all changes together or none;
6. log created, updated, unchanged and disabled counts without dumping schemas.

## 7. Read API

`GET /api/v1/node-catalog?runtimeVersion=X.Y.Z`

- Bearer authentication and route permission are required.
- Missing/malformed `runtimeVersion`: `422`.
- Success: `200`, global `{ data, statusCode }` envelope, `ETag: "<catalogHash>"`.
- Matching strong/weak `If-None-Match`, a matching value in a comma-separated list, or `*`: `304`.
- No authentication: `401`; authenticated role without permission: `403`.
- Return only non-deleted `ACTIVE` nodes compatible with the requested Runtime version.
- Response data is `{ catalogHash, nodes }`; each node exposes all Studio/runtime contract fields
  and expanded capability metadata, but no database IDs, status or publication metadata.
- No pagination while the system catalog remains intentionally small.

Permission `GET /api/v1/node-catalog` is assigned to Admin and Publisher without replacing their
existing permissions.

## 8. Acceptance evidence and remaining integration gate

Historical verification (11-node catalog): source-contract tests, publication behavior, compatibility/disabled filtering,
serialization, ETag handling, `200` with 11 nodes for `0.1.0`, `422`, `304`, `401`, `403`, and an
empty compatible catalog for `0.0.0`.

The earlier catalog passed focused Backend tests and build.
The mode-specific `editorHints.targeting` source contract has passed source parsing
and focused lint only; tests were not requested, the current full build is blocked
by an unrelated generated Prisma/User selection mismatch, and the updated catalog
has not been seeded or verified against a live database in this change.

Not yet verified: Runtime has a registered executable handler for every returned
`runtimeHandler`. Check this at the Runtime integration phase before claiming end-to-end execution.
