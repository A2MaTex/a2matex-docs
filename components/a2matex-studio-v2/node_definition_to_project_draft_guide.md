# Studio Node Definition to Project Draft Guide

## Overview

Studio treats the Backend Node Catalog as the authoring contract. It uses a
Node Definition to build the palette and inspector and saves only instance
values in the Editor Graph. Studio must never copy Node Definition schemas into
a Project Draft. Runtime IR compilation is a separate boundary described only
where its contract is already finalized.

```mermaid
flowchart LR
    C[Backend Node Catalog] --> F[Studio generates node form]
    F --> U[User selects mode and enters values]
    U --> D[Editor Graph in Project Draft]
    D --> V[Backend and local validation]
    V -->|Valid and executable| R[Studio compiles Runtime IR]
    V -->|Invalid or unsupported| E[Show diagnostics and block run or submit]
```

## 1. Load and identify nodes

Fetch `GET /node-catalog?runtimeVersion=X.Y.Z` and cache the response by
`catalogHash`. Match a Project Draft node to its definition through `node.type`.
Do not persist the definition, icon, schemas, runtime handler, or capability
metadata in that Editor Graph node.

Use the definition as follows:

- `name`, `description`, and `category`: palette and inspector;
- `configSchema`: static configuration fields;
- `inputSchema`: ordinary binding fields;
- `outputPorts`: connection handles;
- `editorHints.targeting`: mode-dependent targeting form, when present.

Studio owns icons, colors, field layout, port labels, and target-mode labels.
Render one static connector per `outputPorts.ports` entry in array order. `STOP`
is always available; expose `ERROR_BRANCH` only when the list contains `error`.

## 2. Ordinary node conversion

For a node without `editorHints.targeting`, generate controls from
`configSchema` and `inputSchema`. Save static config values in `node.config`.
Save input values as binding-source objects in `node.bindings`; a direct value
uses `{ "kind": "LITERAL", "value": ... }`.

The Project Draft node keeps only instance data:

```json
{
  "id": "delay_01",
  "type": "flow.delay",
  "config": {},
  "bindings": {
    "durationMs": { "kind": "LITERAL", "value": 1000 }
  },
  "errorPolicy": "STOP",
  "editor": { "x": 420, "y": 180 }
}
```

## 3. Targeting node conversion

When `editorHints.targeting` exists:

1. Render the selector from `targeting.modes` in array order.
2. Save the selected `mode` as `config.targetMode`.
3. Render only that mode's `bindingSchema`; Studio owns the visual field order.
4. Save only the selected mode's values in `bindings`.
5. Require strict literal bindings for the current targeting contract.
6. On a mode change, remove every binding owned by the previous mode before
   writing the new values.

Example selected percentage Tap:

```json
{
  "id": "tap_login",
  "type": "android.tap",
  "config": { "targetMode": "SCREEN_PERCENT" },
  "bindings": {
    "xPercent": { "kind": "LITERAL", "value": 50 },
    "yPercent": { "kind": "LITERAL", "value": 75 }
  },
  "errorPolicy": "STOP",
  "editor": { "x": 660, "y": 180 }
}
```

This record must not also contain `x`, `y`, or `selector_id`. The mode schemas
remain in the Node Catalog and are not copied into the draft.

## 4. Mode switching

Before changing a targeting mode, collect the union of binding names from all
`targeting.modes`. Remove those names from the node bindings, set the new
`config.targetMode`, and initialize only the new mode's form state. Keep future
non-target bindings untouched if a targeting node later defines any.

A targeting edit makes cached `runtimeIr`, validation results, and test results
stale. Recompile and revalidate the same immutable local snapshot before a test
run, cloud sync that carries compiled IR, or submission.

## 5. Runtime IR conversion

Copy `config.targetMode`. Convert each Editor Graph literal binding:

```json
{ "kind": "LITERAL", "value": 50 }
```

into the Runtime IR input source:

```json
{ "source": "LITERAL", "value": 50 }
```

Use the Node Definition for `runtimeHandler`, default timeout, capabilities,
and ports. Copy the node instance's selected `errorPolicy` only after validating
it against the definition's ports: `STOP` is always valid and `ERROR_BRANCH`
requires an `error` port. Do not copy editor position, labels, descriptions,
catalog schemas, or unused targeting modes into Runtime IR.

This guide finalizes only the `LITERAL` mapping needed by the current targeting
nodes. The other currently executable mappings preserve their source metadata:

| Editor Graph binding | Runtime IR input |
|---|---|
| `{ "kind": "WORKFLOW_INPUT", "inputName": "name" }` | `{ "source": "WORKFLOW_INPUT", "inputName": "name" }` |
| `{ "kind": "NODE_OUTPUT", "nodeId": "capture", "field": "artifactId" }` | `{ "source": "NODE_OUTPUT", "nodeId": "capture", "field": "artifactId" }` |

Do not invent Runtime IR forms for `FLOW_VARIABLE` or `GLOBAL_VARIABLE`. Their
executable mapping remains owned by the Runtime workflow-binding phase. Studio
must not offer them for executable workflows until that phase is implemented.

## 6. Validation boundary

Studio validation improves feedback but is not authoritative. Backend validates
the Editor Graph against the active stored Node Definitions, and Runtime
validates the compiled IR again before execution. Studio must block test or
submission when the Runtime handshake does not advertise the selected node
handler, even when the draft itself is valid and saveable.

The complete Backend contracts are documented in
[Node Catalog](../a2matex-backend/studio_project_workflow/node_catalog.md),
[Workflow Validation](../a2matex-backend/studio_project_workflow/workflow_validation.md),
and [Project Draft](../a2matex-backend/studio_project_workflow/project_draft.md).
