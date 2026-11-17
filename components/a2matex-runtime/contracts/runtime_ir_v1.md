# Runtime IR v1 Contract

**Status:** Accepted

**Phase:** A2 — Runtime IR v1 contract

## 1. Purpose and ownership

Runtime IR is the canonical executable document consumed by Runtime. The Backend
compiler produces the authoritative Runtime IR for a submitted Workflow Revision.
The Runtime local compiler produces the same contract for Studio Test Run, but
`ProjectDraft.runtimeIr` is only a nullable authoring cache and is never trusted by
Submission or Product Release.

Runtime IR contains execution semantics only. A compiler omits the entire Editor
Graph `editor` object, including canvas coordinates,
labels, and descriptions. The local Editor Graph validator checks that
metadata as part of the authoring document. Device actions use the executable
node inputs. Runtime IR also excludes viewport state, selection, colors,
timestamps, publisher identity, device selection, and execution input values.

The Backend Node Catalog defines node contracts. A runtime-semantic Node Definition change must increase its
`minimumRuntimeVersion`; adding a node type does not by itself change Runtime IR
`formatVersion`.

## 2. Document envelope

Every Runtime IR v1 document contains exactly these fields:

| Field | Contract |
|---|---|
| `formatVersion` | Integer constant `1` |
| `minimumRuntimeVersion` | Strict `major.minor.patch` SemVer required by all nodes |
| `metadata` | Strict object containing `sourceGraphSchemaVersion` |
| `entryNodeId` | ID of the single `flow.start` node |
| `inputSchema` | Reduced schema for per-device workflow inputs |
| `nodes` | Two to 500 executable nodes; array order has no execution meaning |
| `edges` | One to 1,000 control-flow transitions |
| `requiredCapabilities` | Unique capability union derived from the nodes |

Unknown envelope fields are rejected. The maximum serialized UTF-8 size is
1,048,576 bytes and is checked before full decoding.

`metadata` contains exactly:

```json
{
  "sourceGraphSchemaVersion": 1
}
```

The embedded `inputSchema` must equal the frozen Workflow Revision
`inputSchema`. The Backend compiler copies that frozen value and the Submission
scanner rejects inconsistent output.

## 3. Reduced field schema

A2MaTeX does not implement the complete JSON Schema standard. Workflow input
and Node Definition schemas use this reduced vocabulary:

```json
{
  "type": "object",
  "properties": {},
  "required": [],
  "additionalProperties": false
}
```

Object rules:

- `type` is exactly `object`.
- `properties` contains at most 100 fields.
- `required` contains unique field names that exist in `properties`.
- `additionalProperties` is exactly `false`.
- Properties are flat; nested objects and arrays are unsupported.

Supported field types:

| Type | Optional constraints |
|---|---|
| `string` | `minLength`, `maxLength` |
| `integer` | `minimum`, `maximum` |
| `number` | `minimum`, `maximum` |
| `boolean` | None |

String lengths are integers from zero to 10,000. Integer values and integer
bounds remain inside JavaScript's safe range:
`-9007199254740991..9007199254740991`.
`number` accepts finite JSON numbers, including integers and decimals. Runtime normalizes numeric
values to its floating-point comparison representation.

System-owned Node Definitions may use an approved fixed `pattern`, such as the
Android package-name pattern. Publisher-authored workflow inputs cannot add
arbitrary patterns in v1.

Unsupported features include non-finite numbers, nulls, defaults, enums, unions,
schema composition, recursive schemas, arbitrary formats, and implicit type
conversion.

## 4. Names and identifiers

Node and edge IDs use:

```regex
^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$
```

Field names use:

```regex
^[a-z][a-zA-Z0-9]{0,99}$
```

Node types, runtime handlers, and capabilities use the Backend pattern and a
maximum length of 255:

```regex
^[a-z][a-z0-9_]*(?:\.[a-z][a-z0-9_]*)+$
```

Strict SemVer uses `major.minor.patch` with no prefix, prerelease, or build
suffix.

## 5. Node contract

Each node contains exactly:

| Field | Contract |
|---|---|
| `id` | Unique node ID retained from the Editor Graph |
| `type` | Node contract identity |
| `runtimeHandler` | Handler registry key; no handler version in v1 |
| `config` | Static object validated by the node's config contract |
| `inputs` | Map from handler input field to one input source |
| `timeoutMs` | Effective per-attempt integer timeout from 1 to 600,000 |
| `retryPolicy` | Strict object containing `maxAttempts` and `delayMs` |
| `errorPolicy` | `STOP` or `ERROR_BRANCH` |

`STOP` is valid for every node. `ERROR_BRANCH` is valid only when the node's
approved contract contains the static `error` output-port key. Runtime derives
this support from the port list and rejects an unsupported choice with
`NODE_ERROR_POLICY_INVALID`; Node Definitions do not duplicate a supported-policy list.

`config` and `inputs` each contain at most 100 properties. Tap, Swipe, and Long
Press require `config.targetMode` as specified by the
[Android targeting contract](android_targeting.md). Flow nodes use the specific
configuration defined in the current node matrix and
[workflow binding contract](workflow_bindings.md). Runtime validates `type`,
`runtimeHandler`, config, inputs, outputs, ports, error policy, and capabilities
against its approved node contract registry.

Every Editor Graph node may provide `timeoutMs` and `retryPolicy`. The compiler
uses `NodeDefinition.defaultTimeoutMs` when `timeoutMs` is absent. A missing
retry field resolves independently to the following default:

```json
{
  "maxAttempts": 1,
  "delayMs": 0
}
```

`maxAttempts` is 1–3 and includes the first attempt. `delayMs` is 0–60,000.
The timeout applies separately to each attempt; `delayMs` is the wait after a
retryable failure and before the next attempt. Compiled nodes always contain
effective timeout and retry values; Runtime does not resolve mutable catalog
defaults at execution time.

An Android operation may retry only after an explicitly retryable failure known
to occur before command dispatch. Runtime never automatically replays an
operation whose side effect is uncertain.

## 6. Input sources

An input source states where Runtime obtains one handler parameter. It is not a
data type.

### `LITERAL`

```json
{
  "source": "LITERAL",
  "value": 100
}
```

The object contains exactly `source` and `value`. The value is a string, finite number, or boolean
and must satisfy the destination input field contract. A destination declared as `integer` still
rejects decimal literals.

### `WORKFLOW_INPUT`

```json
{
  "source": "WORKFLOW_INPUT",
  "inputName": "tapY"
}
```

The object contains exactly `source` and `inputName`. The named property must
exist in `inputSchema` and be type-compatible with the destination. A required
node input may reference only a required workflow input. Values are resolved
independently for each device execution.

### `NODE_OUTPUT`

```json
{
  "source": "NODE_OUTPUT",
  "nodeId": "screenshot_01",
  "field": "artifactId"
}
```

The object contains exactly `source`, `nodeId`, and `field`. The producer and
output field must exist, the output and destination types must match, and the
producer must dominate the consumer: it executes before and on every path that
can reach the consumer.

### `FLOW_VARIABLE`

```json
{
  "source": "FLOW_VARIABLE",
  "name": "lastScreenshot"
}
```

The object contains exactly `source` and `name`. Runtime resolves the
case-sensitive key from the current device execution. An absent value or a
value incompatible with the destination input fails the node and follows its
error policy.

### `GLOBAL_VARIABLE`

```json
{
  "source": "GLOBAL_VARIABLE",
  "name": "bestScore"
}
```

The object contains exactly `source` and `name`. Runtime resolves the
case-sensitive key from the current Run Group. An absent value or a value
incompatible with the destination input fails the node and follows its error
policy.

Runtime IR v1 has no free-form expressions, string interpolation, fallback
values, implicit conversions, or executable command text.

## 7. Edge and graph contract

Each edge contains exactly:

| Field | Contract |
|---|---|
| `id` | Unique edge ID |
| `sourceNodeId` | Existing source node |
| `sourcePort` | Port key declared by the source node contract or resolved node instance |
| `targetNodeId` | Existing target node |

Semantic rules:

- Exactly one `flow.start` exists and matches `entryNodeId`.
- Start has no incoming edge and exactly one `success` edge.
- At least one reachable `flow.end` exists.
- End has no outgoing edge.
- Every node is reachable from Start.
- The graph is acyclic.
- Every ordinary non-End node has exactly one `success` edge.
- `flow.switch` has exactly one outgoing edge for each ordered rule ID and one
  `default` edge; it has no `success` edge.
- `ERROR_BRANCH` requires exactly one `error` edge.
- `STOP` prohibits an `error` edge.
- One source node and source port pair has at most one edge.
- Multiple incoming edges are allowed.
- Execution is sequential within one device; arbitrary parallel branches are
  unsupported.

## 8. Capabilities and compatibility

`requiredCapabilities` is the deduplicated union of capabilities from all node
contracts, preserving the first occurrence while traversing Runtime IR nodes and
each definition's capability list. The publisher cannot add or remove entries.
Runtime recomputes and compares the union before execution.

`minimumRuntimeVersion` is the highest minimum version required by the included
node contracts. Runtime requires all three conditions:

1. `formatVersion` is supported.
2. Installed Runtime SemVer is at least `minimumRuntimeVersion`.
3. Every `type` and `runtimeHandler` pair exists in the handler registry.

## 9. Current node matrix

Notation: `int[a..b]`, `string[a..b]`, and `int>=a` describe reduced-schema
constraints. The Android targeting contract defines the mode-specific inputs
for Tap, Swipe, and Long Press.

| Type / handler | Inputs | Output payload | Ports | Default timeout | Error policies | Capabilities |
|---|---|---|---|---:|---|---|
| `flow.start` | `{}` | `{}` | `success` | 1,000 | `STOP` | None |
| `flow.end` | `{}` | `{}` | None | 1,000 | `STOP` | None |
| `flow.delay` | `durationMs: int[0..300000]` | `durationMs: int>=0` | `success`, `error` | 300,000 | `STOP`, `ERROR_BRANCH` | None |
| `flow.set_variable` | `value: supported scalar source` | `{}` | `success`, `error` | 1,000 | `STOP`, `ERROR_BRANCH` | None |
| `flow.wait_variable` | `{}` | `{}` | `success`, `error` | 600,000 | `STOP`, `ERROR_BRANCH` | None |
| `flow.switch` | `value` plus one operand per binary rule | `{}` | ordered UUID rule IDs, `default`, conditional `error` | 1,000 | `STOP`, `ERROR_BRANCH` | None |
| `utility.log` | `message: string[1..4000]` | `{}` | `success`, `error` | 1,000 | `STOP`, `ERROR_BRANCH` | None |
| `android.launch_app` | `packageName: string[3..255]` with approved package pattern | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.tap` | Mode-specific point | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.swipe` | Mode-specific endpoints and duration | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.long_press` | Mode-specific point and duration | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.input_text` | `text: string[0..10000]` | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.press_back` | `{}` | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.press_home` | `{}` | `{}` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.device.control` |
| `android.screenshot` | `targetMode: FULL_SCREEN, ABSOLUTE_PIXELS, or SCREEN_PERCENT` with [mode-specific inputs](../f_device_commands/f7_screenshot.md#1-node-contract) | `artifactId: string[1..]` | `success`, `error` | 30,000 | `STOP`, `ERROR_BRANCH` | `android.screen.capture` |

Node-specific output payloads remain separate because their business results
differ. A5 defines one common execution-result envelope containing the
node-specific payload under `output`.

`flow.set_variable.config` contains required `name` and explicit `scope`; Studio initializes scope
to `FLOW`, and only `FLOW` or `RUN_GROUP` is valid. Its `value` input accepts
any supported scalar source. `flow.wait_variable.config` contains only required
Global variable `name`; the node's effective `timeoutMs` is its wait deadline.
`flow.switch.config.rules` contains one to 32 ordered unique Studio-generated UUIDs and the accepted
numeric, string, and boolean operators. Binary rule operands compile from `bindings[rule.id]`; unary
rules have no operand input. It resolves `inputs.value` once, evaluates rules from first to last,
follows the first matching UUID rule port, and otherwise follows `default`.

## 10. Compilation contract

A compiler receives one Editor Graph, its Input Schema, and an immutable set of
Node Definition semantics. The Backend compiler uses the exact definitions
frozen for the Workflow Revision. The Runtime local compiler uses the
definitions installed in its C7 compilation registry. Studio supplies neither
trusted Node Definitions nor compiler defaults. Neither compiler reads mutable
catalog rows during compilation or mutates its inputs.

It constructs the envelope as follows:

- set `formatVersion` to `1` and copy Editor Graph `schemaVersion` into
  `metadata.sourceGraphSchemaVersion`;
- copy `entryNodeId`, Input Schema, and edges in their existing array order;
- derive `minimumRuntimeVersion` from the referenced Node Definitions;
- compile nodes in Editor Graph array order;
- derive `requiredCapabilities` by first-occurrence deduplication.

For every node, the compiler:

1. copies `id`, `type`, validated `config`, and `errorPolicy`;
2. resolves `runtimeHandler` and default timeout from its frozen Node Definition;
3. maps every binding `kind` to the corresponding Runtime IR input `source`;
4. writes `timeoutMs = node.timeoutMs ?? definition.defaultTimeoutMs`;
5. writes `retryPolicy.maxAttempts = node.retryPolicy.maxAttempts ?? 1`;
6. writes `retryPolicy.delayMs = node.retryPolicy.delayMs ?? 0`; and
7. omits the complete Editor Graph `editor` object and every catalog-only field.

Set Variable, Wait Variable, Switch, variable sources, targeting modes, and
dynamic Switch ports follow Sections 6, 7, and 9. Any validation or compilation
error produces no Runtime IR result; warnings may accompany a complete IR. The
compiler never emits partial IR.

## 11. Validation pipeline

Runtime validates the complete document before creating an active execution:

1. Enforce the UTF-8 byte limit.
2. Strictly decode the envelope and nested structures.
3. Validate reduced schemas, identifiers, sizes, and policy bounds.
4. Resolve each node contract and validate config, input names, handler, ports,
   policies, and literal values.
5. Validate edge endpoints, Start/End rules, reachability, and acyclicity.
6. Type-check every input source, including node-output producer dominance and
   variable destination compatibility.
7. Recompute capabilities and minimum Runtime compatibility.

Validation failure creates no active execution and dispatches no device
command. A6 defines the eventual error envelope and codes.

## 12. Reference document

[`runtime_ir_v1.example.json`](./runtime_ir_v1.example.json) is the canonical
valid A2 example. Runtime implements the contract with strict Go types and
explicit validation; Studio and Backend implement their corresponding strict
TypeScript contracts. No generic or custom schema interpreter is required.

A8 owns the [shared valid and invalid fixture suite](./contract_fixtures_v1.md)
used to verify that the Go and TypeScript implementations accept and reject the
same contract cases.
