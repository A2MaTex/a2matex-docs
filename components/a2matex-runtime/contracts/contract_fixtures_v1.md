# Runtime Contract Fixtures

**Status:** Accepted

**Phase:** A8 — Contract fixtures

## 1. Responsibility

A8 supplies identical, portable input bytes and expected classifications for
contracts implemented independently in Go and TypeScript. Its purpose is to
detect a concrete failure: Studio or Backend accepts a document that Runtime
rejects, or a client decodes an event/API payload differently from Runtime.

A8 does not implement a validator or prove that any current implementation is
conformant. It provides the shared evidence inputs that C6 and later contract
tests must execute.

```text
accepted contract
      -> immutable fixture bytes + manifest expectation
      -> TypeScript contract test
      -> Go contract test
      -> compare classifications and required codes
```

## 2. Findings from the previous documents

Earlier plans required “shared fixtures” but did not define their owner,
format, expected result, or synchronization mechanism. The inspected Backend
branch currently contains only Node Catalog definition fixtures; its documented
Editor Graph validation fixtures are not present on this checkout. Runtime has
no contract tests yet. The old fake-ADB examples test process/device adapters,
not cross-language JSON contracts, and therefore do not belong to A8.

The existing [`runtime_ir_v1.example.json`](./runtime_ir_v1.example.json) is a
useful accepted Runtime IR case and remains part of the fixture set rather than
being duplicated.

## 3. Canonical package

The canonical package is:

```text
contracts/
├── runtime_ir_v1.example.json
├── fixtures/v1/              # historical accepted set
└── fixtures/v2/              # current set
    ├── manifest.json
    ├── editor_graph/
    ├── runtime_ir/
    ├── event_error/
    └── local_api/
```

[`fixtures/v2/manifest.json`](./fixtures/v2/manifest.json) is the current
machine-readable index. Fixture set v2 removes the redundant fixed `mode` from
the A7 runtime-information response; v1 remains unchanged for historical
consumers.

All paths in it are relative to the `contracts` directory. Every payload has a
SHA-256 digest so a vendored copy cannot be reformatted or changed silently.

The plan repository owns the canonical bytes. Backend, Studio, and Runtime
vendor the applicable files into test-only directories when their consuming
tests are implemented. They must not depend on a sibling checkout path, bundle
these fixtures in a release binary, or create implementation-specific edited
copies under the same fixture ID.

Every accepted version directory is immutable. A changed payload, digest, or
expectation creates a new fixture-set directory and version; it does not change
Runtime IR `formatVersion`, Local API `/v1`, or a node-handler version.

## 4. Manifest contract

The manifest contains:

| Field | Rule |
|---|---|
| `fixtureSetVersion` | Positive integer matching the directory version |
| `contractSet` | Stable label for the accepted Phase A contract set |
| `cases` | Ordered list of unique cases |

Every case contains:

| Field | Rule |
|---|---|
| `id` | Stable dotted identifier; tests report this value |
| `contract` | `EDITOR_GRAPH_V1`, `RUNTIME_IR_V1`, `EXECUTION_EVENT_V1`, `RUNTIME_ERROR_V1`, or `LOCAL_API_V1` |
| `path` | Safe relative path under `contracts`; no absolute path or `..` segment |
| `sha256` | Lowercase digest of the exact payload bytes |
| `expect` | Contract-specific classification and only the comparison fields required below |

Harness-only values such as `violatedRule` and `shape` label the assertion.
They are not new Runtime error codes or wire fields.

## 5. Comparison rules

### Editor Graph

The validator must produce the manifest `VALID` or `INVALID` classification.
For invalid cases, compare the ordered diagnostic-code list. Message wording,
localized presentation, and internal parser errors are not fixture assertions.

### Runtime IR

Strict decoders and semantic validators must accept or reject the document as
listed. `violatedRule` identifies the tested contract rule; A8 does not invent
a G1 diagnostic-code catalog.

### ExecutionEvent and RuntimeError

Strict decoders must accept valid cases and reject invalid cases. Accepted
events must also resolve to the listed closed event type. Unknown fields remain
rejected.

### Local API and SSE

Request/response DTO decoders must accept or reject the listed shape. The
invalid start request maps to the specified HTTP status and A6 error code when
tested through HTTP. The SSE case must parse its `id`, `event`, and `data`, then
produce JSON exactly equal to the referenced ExecutionEvent fixture.

JSON object property order is not compared after parsing. Array order, types,
values, omitted fields, explicit nulls, and unknown-field rejection remain
significant.

## 6. Fixture inventory

The 19 cases are deliberately representative rather than exhaustive.

| Family | Cases | Boundary protected |
|---|---:|---|
| Editor Graph | 3 | Valid minimal graph, strict unknown-field rejection, deterministic cycle diagnostic |
| Runtime IR | 4 | Rich valid reference, unsupported format, unknown field, recomputed capability mismatch |
| Events and errors | 7 | All four A6 event variants, required correlation, valid error, unknown error field |
| Local API and SSE | 5 | Compatibility response, strict start request, unknown start field, snapshot, SSE framing |

This suite includes one meaningful valid case and high-risk rejection cases at
each shared boundary. Adding dozens of nearly identical bound checks would make
the fixture package harder to review without replacing implementation-owned
unit tests.

## 7. What remains outside A8

A8 intentionally excludes:

- Node Catalog creation/update fixtures, which remain Backend-owned;
- A4 typed device commands, which never cross the Runtime API;
- every numeric/string boundary and every diagnostic path/message;
- oversized payload files generated only to test byte limits;
- A5 transition, aggregation, checkpoint, and crash-recovery scenarios;
- authentication, CORS, persistence, transaction, retention, and SSE
  backpressure integration tests;
- ADB output, process-runner, and real-device fixtures;
- screenshot binary fixtures and artifact-store behavior; and
- Production package, authorization, license, connector, advanced-node, and
  handler-version cases.

Those belong to their owning phase tests. A fake ADB fixture can prove parser
or adapter behavior but cannot satisfy a real-device acceptance gate.

## 8. Phase connections

| Consumer | Required use |
|---|---|
| C1–C6 | Run Editor Graph cases against Backend and Runtime; compare classification and ordered diagnostic codes |
| G1 | Run Runtime IR cases before admission implementation is approved |
| D3/G/H | Decode A6 event/error cases and preserve exact closed variants |
| B6/C5/E7/I1 | Decode or emit applicable A7 API cases |
| I1 | Parse the SSE frame and compare its data to the event fixture |
| J | Verify vendored SHA-256 digests and execute all applicable cases in CI |

Studio, Backend, and Runtime keep independent implementation code. Passing the
same fixtures demonstrates agreement only for these listed cases; it does not
prove the validators are equivalent for every possible document.

## 9. Change procedure

1. Change the owning contract first and review its effect across consumers.
2. Create a new versioned fixture directory rather than mutating accepted v1.
3. Recalculate payload digests and update the new manifest.
4. Vendor the new set into each affected implementation.
5. Make old and new expectations explicit during any compatibility window.
6. Remove an old set only after no supported producer or consumer needs it.

## 10. Checkout condition

A8 documentation is complete when every manifest entry resolves under the
contracts directory, every digest matches, all JSON payloads parse, case IDs
and paths are unique, and the SSE frame matches its referenced event fixture.

Cross-language conformance remains unverified until Backend/Studio TypeScript
and Runtime Go tests consume this package in C6, G1, I1, and J.
