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
- The current supported error policies are `STOP` and `ERROR_BRANCH`.

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
| Studio/error     | `editorHints`, `supportedErrorPolicies`                       |
| Sync/audit       | `contentHash`, `createdAt`, `updatedAt`, `deletedAt`          |

Database invariants: unique `type`, positive timeout, 64-character lowercase hex hash, all six JSON
fields are objects, and index `(status, type)`.

### `NodeDefinitionCapability`

Joins a current definition to a capability. The pair `(nodeDefinitionId, capabilityId)` is unique.
Foreign keys use the database default `NO ACTION` behavior.

## 4. Definition contract

Each definition declares:

- identity/display: `type`, `name`, `description`, `category`;
- execution: `runtimeHandler`, strict `minimumRuntimeVersion`, bounded positive timeout;
- three object schemas with `type: "object"`, `properties`, `required`, and
  `additionalProperties: false`;
- unique output ports from `success`, `error`, `true`, `false`, `body`, `completed`;
- editor icon and ordered fields;
- error policies and capability codes.

Required fields must exist in `properties`; editor fields must exist in config or input; duplicate
ports, fields, policies and capabilities are invalid. `ERROR_BRANCH` and the `error` port must be
declared together. No executable expression or JavaScript is accepted.

### Agreed v1 update — pending implementation

Add optional string `editorHints.documentation` for type-level help. Omission and
the empty string are both valid; Studio hides Help in either case. Keep the short
`description` and `fieldOrder`. Studio maps icon and color from `type` and provides
a fallback for unmapped types. Switch Studio to that mapping before removing the
existing `editorHints.icon` from the Backend definition and response. No backend
color field, new database column, or Runtime IR change is needed. Catalog hashing
and ETag continue to cover the returned catalog content.

## 5. Initial catalog

| Category | Node types                                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Flow     | `flow.start`, `flow.end`, `flow.delay`                                                                                                       |
| Utility  | `utility.log`                                                                                                                                |
| Android  | `android.launch_app`, `android.tap`, `android.swipe`, `android.input_text`, `android.press_back`, `android.press_home`, `android.screenshot`, `android.long_press` |

The Backend source catalog contains 12 definitions. Control flow beyond delay is deferred.

### Planned targeting modes and Long Press

The seed includes Long Press. Catalog presence does not indicate Runtime handler
availability. Tap, Swipe, and Long Press expose a list of mode names in
`editorHints.targetModes`; their flat `inputSchema` and `fieldOrder` still describe
pixel inputs and duration where applicable. The percentage and selector fields
below describe the planned forms; they are not published as input schemas.

| Node type | Absolute pixels (`ABSOLUTE_PIXELS`) | Screen percentage (`SCREEN_PERCENT`) | Element ID (`SELECTOR_ID`) | Additional input |
|---|---|---|---|---|
| `android.tap` | `x`, `y` (existing) | `xPercent`, `yPercent` | `selector_id` | None |
| `android.swipe` | `startX`, `startY`, `endX`, `endY` (existing) | `startXPercent`, `startYPercent`, `endXPercent`, `endYPercent` | Unsupported | Existing gesture `durationMs` in both modes |
| `android.long_press` (new) | `x`, `y` | `xPercent`, `yPercent` | `selector_id` | Required positive integer hold `durationMs`; execution bounds to be agreed with the handler |

- Percentages are integers from 0 to 100, measured from the top-left of the
  current device screen in its current orientation. Resolve each axis against
  its own dimension: `round(percent × (dimension - 1) / 100)`. Scaling does
  not compensate for controls that move after a layout or orientation change.
- `selector_id` is the non-empty, full Android UI XML `resource-id`, for
  example `com.example.app:id/login_button`. At execution, resolve an exact
  match in the current hierarchy and act at the center of its visible bounds.
  Missing matches, multiple matches, or unusable bounds fail the action;
  never fall back to coordinates. The authoring XML is not an execution input.
- Long Press is an Android device-control action. Its handler is planned;
  the existing Input Text remains a focused-field action and can follow Tap.
  No input changes for the other nine existing nodes. Scroll is out of scope.
- The [Editor Graph rules](workflow_validation.md#planned-targeting-in-editor-graph-v1)
  describe the planned draft representation. Runtime IR remains unchanged.

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

The 12-definition source catalog passed focused Backend tests and build; the updated
catalog has not been seeded or verified against a live database in this change.

Not yet verified: Runtime has a registered executable handler for every returned
`runtimeHandler`. Check this at the Runtime integration phase before claiming end-to-end execution.
