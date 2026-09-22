# Workflow Authoring and Runtime Semantics

## 1. Workflow representation

A workflow is a directed graph made of nodes, ports, and edges. The authoring
graph may include visual metadata, but security review and Runtime execution use
normalized semantic data.

A graph node should reference source rather than embed all source and binary data:

```json
{
  "id": "0199...",
  "type": "custom.integration",
  "definitionVersion": "1.0.0",
  "sourceRef": "source-789",
  "sourceHash": "sha256:...",
  "config": {},
  "inputs": {},
  "activationPolicy": "ALL_SUCCESS"
}
```

Assets are object references with hashes. They are not stored inside graph JSON.

## 2. Stable node identity

Studio creates an immutable UUID when a node is added.

- Moving a node does not change its ID.
- Renaming a node does not change its ID.
- Editing config or code does not change its ID.
- Copying a node creates a new ID.
- Delete then Undo restores the previous ID.
- Delete then create a new node creates a new ID.
- Duplicate or malformed IDs are rejected at submission.

If a user edits `workflow.json` manually, the server interprets a disappeared ID
as deleted and a new ID as added. Replacing code while preserving the node ID is
still detected by the source hash.

## 3. Diff model

For each node, maintain separate comparison dimensions:

- `source_hash` - custom source code.
- `semantic_hash` - type, version, config, mapping, policy; excludes layout.
- `visual_hash` - position, size, color, viewport.
- `capability_hash` - capability, domain, credential slot, and side-effect set.

For a 300-node graph, a map-by-ID comparison is sufficient:

```text
ID only in new graph -> added
ID only in old graph -> deleted
ID in both:
  source_hash changed     -> code modified
  semantic_hash changed   -> behavior/config modified
  capability_hash changed -> security-relevant change
  only visual_hash changed -> visual-only change
```

Visual-only changes should not trigger the same security workload as source or
capability changes.

## 4. Node taxonomy

| Node type | Purpose | UAT |
|---|---|---|
| Start/Trigger | Entry point and initial input | Required |
| System Node | Versioned platform operation | Required; about 6-10 initially |
| Transform Node | Map, filter, format, validate | Required |
| Custom TypeScript Node | Restricted custom logic | Required |
| Custom Integration Node | HTTP/credential/capability through SDK | Required |
| Merge Node | Join or wait for upstream branches | Required |
| Loop Controller | Controlled repetition | Required |
| Native Custom Node | Publisher native code | After UAT |

## 5. Port and data rules

- Every node has versioned input, config, and output schemas.
- Edge source and target ports must be compatible.
- Required input must be mapped, defaulted, or deliberately skipped according to
  the target node's missing-input policy.
- Input mapping is evaluated against completed upstream output and execution
  context.
- Runtime validates node input before invoking the handler and validates output
  before publishing it to downstream nodes.

## 6. Conditional edges

Conceptual structure:

```json
{
  "id": "edge-123",
  "sourceNodeId": "check-order",
  "sourcePort": "success",
  "targetNodeId": "send-email",
  "targetPort": "input",
  "condition": {
    "type": "expression",
    "expression": "{{ source.output.total > 1000000 }}"
  },
  "priority": 1
}
```

### Branch modes

| Mode | Behavior |
|---|---|
| `FIRST_MATCH` | Execute the first true edge in priority order |
| `ALL_MATCHES` | Execute every true edge |
| `EXCLUSIVE` | Exactly one edge must be true; zero or multiple is an error |
| `DEFAULT` | Execute if no conditional edge matches |

The expression language must be deterministic, bounded, and unable to access
arbitrary Runtime globals. Its exact grammar belongs in SDS.

## 7. Activation policies

A node with multiple upstream dependencies decides when it is ready using an
activation policy.

| Policy | Ready condition |
|---|---|
| `ALL_SUCCESS` | Every required upstream completed successfully |
| `ALL_COMPLETED` | Every required upstream reached a terminal state |
| `ANY_SUCCESS` | At least one upstream succeeded |
| `FIRST_SUCCESS` | First upstream success activates the node |
| `N_OF_M` | Configured number of upstream dependencies satisfy the rule |

“Wait for all previous nodes” belongs to the destination or Merge Node, not
solely to an edge.

### Missing-input policies for UAT

- `FAIL` - fail the node and apply workflow error policy.
- `USE_DEFAULT` - use the schema/configured default.
- `SKIP_NODE` - mark the node skipped and do not invoke its handler.

`WAIT` is not required for UAT because it introduces persistent waiting and
external-event semantics.

## 8. Controlled loops

Arbitrary graph cycles are prohibited. Only a Loop Controller may own a validated
back edge.

```text
Loop Controller --each(item,index)--> Process Item
      ^                                  |
      |-------------continue-------------|

Loop Controller --done(results)--> Next Node
```

### Loop modes

- `FOR_EACH` - iterate a dynamic collection.
- `REPEAT` - iterate a dynamic count.
- `WHILE` - continue while a bounded expression is true.

Example configuration:

```json
{
  "mode": "FOR_EACH",
  "collection": "{{ nodes.fetch.output.items }}",
  "executionMode": "sequential",
  "runtimeHardLimit": 10000,
  "timeoutSeconds": 300,
  "continueOnError": false,
  "collectOutputs": true,
  "maxCollectedOutputBytes": 5242880
}
```

Dynamic input never bypasses a platform safety ceiling:

```text
actualIterations = min(evaluatedIterations, runtimeHardLimit)
```

Exceeding a ceiling produces a resource-limit error. It must not silently create
an unbounded run.

## 9. TypeScript node environment

Publisher code should resemble:

```typescript
import { defineNode, type NodeContext } from "@platform/sdk";

export default defineNode({
  async run(input: Input, context: NodeContext): Promise<Output> {
    const response = await context.http.request({
      method: "POST",
      url: "https://api.example.com/items",
      body: input.items,
    });

    return { status: response.status };
  },
});
```

Allowed import categories for UAT:

- Platform SDK.
- Platform-approved utilities.
- Optionally, restricted project shared modules.

Not allowed:

- Arbitrary npm packages.
- `fs`, `process`, `child_process`, direct sockets, or unrestricted globals.
- `eval`, dynamic import, or dynamic code loading.
- Direct Android/Kotlin API access.

Network must use `context.http`. Camera and Bluetooth must use SDK capability
methods that route through Capability Broker.

## 10. Side effects and God Nodes

A node may perform multiple internal computation steps but should have one main
observable side effect.

Good example:

1. Normalize a request.
2. Append rows to Google Sheets.
3. Parse the response.

Risky example:

1. Send an email.
2. Write a Sheet.
3. Delete a file.
4. Send Telegram.

Multiple unrelated side effects make retries, monitoring, review, and failure
recovery ambiguous.

Build/review data should leave room for:

```text
capabilities
side_effects
timeout
retry_policy
resource_policy
complexity_score
```

## 11. Scheduler lifecycle

Conceptual node states:

```text
PENDING -> READY -> RUNNING -> SUCCEEDED
                      |-----> FAILED
                      |-----> CANCELLED
PENDING/READY ----------------> SKIPPED
```

Conceptual execution states:

```text
CREATED -> VALIDATING -> RUNNING -> SUCCEEDED
                         |-------> FAILED
                         |-------> CANCEL_REQUESTED -> CANCELLED
```

The scheduler must:

1. Validate graph and initial input.
2. Compute nodes whose dependencies and conditions are satisfied.
3. Reserve resource/concurrency capacity.
4. Invoke the correct System or custom handler.
5. Validate output and emit an event.
6. Evaluate downstream edges and readiness.
7. Apply retry, error-edge, skip, or failure policy.
8. Stop scheduling new work after cancellation or terminal failure.

## 12. Development execution

### Run complete workflow

- Uses the same graph semantics as production.
- May use development credentials, test input, and mocked capability adapters.
- Shows input, output, error, duration, and retry per node.

### Run one node with pinned input

- Validates input schema.
- Requires upstream data to be pinned or mocked.
- Runs the real node handler under development sandbox policy.
- Does not pretend that downstream graph behavior was tested.

## 13. Execution analytics

Development logs are per run and may show sanitized details. Production
analytics are aggregate:

```text
Total runs
Success rate
P95 duration
Most failed node
Top error code
Permission-denied rate
Resource-limit rate
```

Payload telemetry is not enabled by default.

## 14. Open decisions for SDS

- Expression language and evaluator.
- JS engine and module format on Android.
- Exact scheduler persistence and crash recovery model.
- Retry/idempotency contract for side-effect nodes.
- Concurrency model for branches and loops.
- Resource accounting accuracy and enforcement mechanism.
- Whether long-running/waiting workflows are supported after UAT.

