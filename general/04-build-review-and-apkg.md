# Build, Review, `.apkg`, and Distribution

## 1. Draft, revision, submission, and artifact are different

| Object | Mutable | Purpose |
|---|---:|---|
| Local save | Yes | Current project work on the Creator's device |
| Server Draft | Yes | Optional current synchronized state for a Registered Project |
| Workflow Revision | No | Immutable graph/source/metadata/asset snapshot |
| Submission | Status changes only | Request to validate and review one Revision |
| Candidate Artifact | No | `.apkg` produced before production signing |
| Published Version | No | Product Version pointing to a signed artifact |

A submission never points at “whatever the Draft currently contains.”

## 2. Snapshot and upload flow

1. Studio validates the current project.
2. Studio freezes graph, source, metadata, and asset hashes.
3. Studio creates a compressed snapshot.
4. Studio calculates the content hash.
5. Snapshot uploads directly or through resumable/multipart object upload.
6. Backend receives the object reference, expected hash, and size metadata.
7. Backend verifies hash, size, file count, expanded size, and compression ratio.
8. A Workflow Revision and Submission are created.

Large submissions should not be buffered as one large JSON body through the main
API service.

## 3. Automated validation

### 3.1 Structural validation

- Graph/schema version is supported.
- Node type and definition version exist.
- Node and edge IDs are unique and valid.
- Edge ports exist and types are compatible.
- Required inputs are mapped or have valid missing-input policy.
- Entry point is valid.
- Unreachable nodes are reported.
- Arbitrary cycles are rejected.
- Loop edges return to the correct Loop Controller.
- Activation and branch policies are valid.

### 3.2 Source validation

- TypeScript compilation and lint.
- Forbidden import detection.
- `eval`, dynamic import, and dynamic code loading detection.
- Obvious infinite-loop and dangerous-pattern checks.
- Secret literal scanning.
- Direct network/native access detection.
- Code, AST, and complexity limits.
- Dynamic or undeclared domain detection.
- Capability and credential-use extraction.

Automation reduces known violations. It does not prove the code is completely
safe, which is why human review remains required.

## 4. Capability extraction

Build Service derives capabilities from:

- System Node definitions.
- `context.http` calls and configured domains.
- `context.camera` and Bluetooth calls.
- File and notification SDK calls.
- Credential bindings.
- Graph and node resource behavior.

The build compares declared and detected capability sets. A detected capability
outside the declaration causes validation failure.

## 5. Build pipeline

1. Parse `workflow.json`.
2. Validate graph structure and semantics.
3. Resolve System Node versions.
4. Load each Custom Node source.
5. Type-check source using platform-controlled configuration.
6. Resolve approved SDK and utility imports.
7. Reject forbidden imports and patterns.
8. Transpile to the Runtime-supported module representation.
9. Calculate source hashes.
10. Compile the graph into Workflow IR.
11. Bind module paths to immutable node IDs.
12. Generate capability manifest and domain allowlist.
13. Generate node lock and content hash list.
14. Package schemas, approved assets, and UI metadata.
15. Create the candidate `.apkg`.
16. Run sandbox tests.

## 6. Sandbox testing

The sandbox uses:

- Fake or test credentials.
- Mock HTTP responses.
- Fake Camera and Bluetooth results.
- Test input fixtures.
- Timeout and resource accounting.

It should detect:

- Handler crash.
- Request outside allowlist.
- Output or log overflow.
- Non-terminating loop within detectable limits.
- Missing error path.
- Capability mismatch.
- Resource-limit violation.

## 7. Human review workflow

Reviewer sees:

- Product and version metadata.
- Visual graph.
- Node responsibilities and side effects.
- Source and source diff.
- Semantic graph diff.
- Capability, domain, and credential delta.
- Automated findings and risk summary.
- Sandbox results.
- Resource warnings.

Decisions:

- `APPROVE`.
- `REJECT`.
- `REQUEST_CHANGES`.
- `ESCALATE`.

Every decision records actor, timestamp, reason, policy version, Revision ID, and
candidate artifact hash.

## 8. Submission state model

```text
DRAFT
  -> SUBMITTED
  -> VALIDATING
  -> BUILDING
  -> SANDBOXING
  -> IN_REVIEW
      -> CHANGES_REQUESTED
      -> REJECTED
      -> ESCALATED
      -> APPROVED
          -> SIGNING
          -> PUBLISHED
```

Build or signing infrastructure failure is not the same as product rejection.
Those states must remain distinguishable for retry and support.

## 9. Signing invariant

Signing Service signs only the exact candidate hash referenced by an approved
Review Decision.

```text
approved_hash == fetched_candidate_hash == signed_artifact_hash
```

If the artifact changes by one byte after review, verification must fail. The
normal Build Worker should not possess the production signing key.

## 10. Logical `.apkg` structure

```text
product-1.2.0.apkg
├── manifest.json
├── workflow.ir
├── scripts/
│   ├── <node-id>.bin
│   └── <node-id>.bin
├── schemas/
├── assets/
├── node-lock.json
├── permissions.json
├── hashes.json
└── signature.sig
```

### `manifest.json`

Contains at least:

- Product ID and version.
- Runtime version range.
- Entry node.
- Capability set.
- Domain allowlist.
- Credential slots.
- Package/schema format versions.

### `workflow.ir`

Contains normalized executable instructions. Example:

```json
{
  "nodes": [
    {
      "id": "node-123",
      "handler": {
        "type": "custom-script",
        "module": "scripts/node-123.bin"
      }
    },
    {
      "id": "node-456",
      "handler": {
        "type": "system",
        "key": "google-sheets.append",
        "version": "1.0.0"
      }
    }
  ],
  "edges": []
}
```

`.apkg` is a platform-defined container. It is not an executable and does not
install a language runtime per product.

## 11. Launcher install flow

1. Store download in temporary private storage.
2. Enforce compressed package size.
3. Verify expected content hash.
4. Verify platform signature and signer status.
5. Verify package/product/version identity.
6. Verify supported package and Runtime versions.
7. Check revocation state available to the Launcher.
8. Parse manifest and display capability/credential summary.
9. Move verified package atomically into installed storage.
10. Mark the product version Installed.

On failure, Launcher deletes or quarantines the candidate download and preserves
the currently active version.

## 12. Update and rollback

- A new Product Version is a new immutable package.
- Launcher verifies it separately before activation.
- Increased capability, domain, or credential scope requires renewed disclosure
  and Customer consent. It is never automatically granted from an older version.
- Activation should be atomic at the active-version reference.
- UAT should retain the two previous `.apkg` versions where storage policy allows.
- Rollback must still enforce current revocation and license rules.

## 13. Baseline package limits

| Item | UAT baseline |
|---|---:|
| Graph JSON | 10 MB |
| Total source | 10 MB |
| One Custom Node source | 256 KB |
| One asset | 10 MB |
| Total assets | 50 MB |
| Compressed submission | 50 MB |
| Expanded submission | 100 MB |
| File count | 1,000 |
| Compression ratio | Maximum 20:1 |

These are configurable safety defaults, not permanent product promises.

## 14. Open decisions for SDS/security design

- Exact archive format and canonical hashing algorithm.
- Reproducible-build guarantees and toolchain identity.
- Signing key service, rotation, compromise recovery, and certificate chain.
- SBOM format and whether it is embedded or stored beside the artifact.
- Emergency revocation propagation.
- Two-person approval for high-risk products.
- Artifact encryption or anti-reverse-engineering measures after UAT.

