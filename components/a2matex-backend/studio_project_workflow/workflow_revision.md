# Workflow Revision Feature Spec

**Status:** Backend core implemented; Submission transaction integration pending

**Phase:** `a2matex-be/phase5.md`

**Owner:** Backend; created only by the future Web submission use case

## 1. Purpose and boundary

`WorkflowRevision` is the immutable Backend snapshot of one exact `ProjectDraft` version. It is the
freeze point between mutable Studio authoring and the Submission, scan, review, package, and release
pipeline.

```text
Studio -> mutable ProjectDraft
Web submission -> immutable WorkflowRevision -> Submission -> scan/review -> ProductRelease
```

Studio does not create a revision directly. The future Submission application service calls the
Workflow Revision service inside the same database transaction that creates the Submission and its
outbox event. A successfully committed revision must therefore belong to exactly one committed
submission; a failed transaction must leave neither record behind.

This module owns:

- loading and freezing the exact requested draft version;
- deriving a complete historical Node Definition manifest from the current system catalog;
- deriving the required capability manifest;
- calculating deterministic provenance and revision hashes;
- persisting immutable revision content and exposing it to trusted Backend consumers.

This module does not own:

- Studio draft editing, optimistic save conflicts, or draft history;
- Web self-review, editable Product presentation/media, Release metadata, or Submission status;
- asynchronous cloud validation, policy scanning, review, package building, or encryption;
- Asset upload/storage lifecycle;
- Launcher download authorization or Runtime execution.

## 2. Proposed MVP decisions for review

- There is no public Workflow Revision create, update, archive, or delete endpoint.
- Creation is an internal `createFromDraft` use case invoked by Submission processing.
- A revision is append-only. It has no `updatedAt`, `deletedAt`, update repository method, or delete
  repository method.
- The caller supplies `projectId` and `sourceDraftVersion`; the Backend derives all content, hashes,
  manifests, timestamps, and actor fields.
- The source Project must be accessible to the active Publisher owner and must be `ACTIVE`.
- The source draft must exist, remain current, and have exactly the requested version at the moment
  it is locked for submission.
- `runtimeIr` must be a non-null JSON object. The module does not compile or repair it.
- One Project draft version can produce at most one revision. A publisher must save a new draft
  version before creating a materially new submission.
- Draft `validationSummary` and `testSummary` are frozen as Studio-provided advisory evidence. They
  never authorize submission, approval, packaging, or release.
- Cloud validation and scan results are stored by the Submission pipeline, not written back into the
  immutable revision.
- The current Node Catalog has mutable definitions and no historical node versions. Consequently,
  the revision embeds the full contract of every distinct referenced Node Definition instead of
  storing only `type + contentHash`.
- Initial asset-free workflows use an empty asset list in the hash contract. Asset-bearing submission
  remains unsupported until the Asset module can freeze `alias + assetHash` references.

## 3. Data model

```text
WorkflowRevision
  id UUID PK
  projectId UUID FK Project, required
  sourceDraftVersion INTEGER, required, positive
  sourceDraftContentHash VARCHAR(64), required, lowercase SHA-256
  editorGraph JSONB object, required
  runtimeIr JSONB object, required
  inputSchema JSONB object, required
  nodeDefinitionManifest JSONB object, required
  capabilityManifest JSONB object, required
  validationSummary JSONB object, required
  testSummary JSONB object, required
  contentHash VARCHAR(64), required, lowercase SHA-256
  createdAt TIMESTAMPTZ, required
  createdById UUID FK User, required

  UNIQUE(projectId, sourceDraftVersion)
```

`sourceDraftContentHash` preserves the Backend-computed hash of the draft selected by the publisher.
`contentHash` identifies the larger frozen revision snapshot after Backend-derived manifests and
assets are added. The hashes are deliberately different contracts.

There is no foreign key to a historical `ProjectDraft` version because the current Draft model keeps
one mutable row rather than version rows. Provenance is represented by `projectId`,
`sourceDraftVersion`, and `sourceDraftContentHash`.

Every JSON column must be an object at the database boundary. Both hashes must match
`^[0-9a-f]{64}$`. Foreign keys use `ON DELETE NO ACTION`. Identical revision content across different
Projects is valid, so `contentHash` is not globally unique.

### Future asset relation

After the Asset module exists:

```text
WorkflowRevisionAsset
  id UUID PK
  workflowRevisionId UUID FK WorkflowRevision, required
  assetId UUID FK Asset, required
  alias VARCHAR(255), required
  assetHash VARCHAR(64), required, lowercase SHA-256
  createdAt TIMESTAMPTZ, required

  UNIQUE(workflowRevisionId, alias)
  UNIQUE(workflowRevisionId, assetId)
```

The first Workflow Revision migration must not create this relation before the referenced Asset
model exists. Until then, revision creation accepts only workflows for which the submission contract
contains no asset references and hashes `assets: []`.

## 4. Frozen snapshot contracts

### Node Definition manifest

The Backend extracts distinct node types from `editorGraph`, loads them in one batched query, rejects
missing, disabled, or soft-deleted definitions, and stores entries sorted lexicographically by
`type`.

```json
{
  "schemaVersion": 1,
  "definitions": [
    {
      "type": "android.tap",
      "name": "Tap",
      "description": "Tap an Android screen coordinate",
      "category": "Android",
      "runtimeHandler": "android.tap",
      "minimumRuntimeVersion": "0.1.0",
      "defaultTimeoutMs": 10000,
      "configSchema": {},
      "inputSchema": {},
      "outputSchema": {},
      "outputPorts": {},
      "editorHints": {},
      "supportedErrorPolicies": {},
      "capabilityCodes": ["android.device.control"],
      "contentHash": "64-character-lowercase-sha256"
    }
  ]
}
```

Database IDs, publication timestamps, mutable status, and soft-delete metadata are excluded. The
manifest contains the complete contract needed to audit and later package the submitted workflow.
The stored `contentHash` of each definition is copied as its catalog-publication fingerprint. The
current publisher hash includes the source capability-array order, while relational capability rows
have no durable ordering, so this module must not pretend it can recompute that hash exactly. The
Workflow Revision `contentHash` independently seals the actual full manifest stored in the revision.

### Capability manifest

The Backend derives the union of capability codes from the frozen Node Definition entries, removes
duplicates, and sorts codes lexicographically.

```json
{
  "schemaVersion": 1,
  "capabilities": ["android.device.control", "android.screen.capture"]
}
```

The publisher cannot supply or remove capabilities. Capability policy evaluation belongs to the
later scanner; this module only freezes the exact derived requirement set.

## 5. Deterministic revision hash

The module recursively sorts object keys, preserves scalar values, and preserves array order except
for Backend-derived arrays whose ordering is defined below. It then computes lowercase SHA-256 over
the UTF-8 canonical JSON of exactly this object:

```json
{
  "hashVersion": 1,
  "editorGraph": {},
  "runtimeIr": {},
  "inputSchema": {},
  "nodeDefinitionManifest": {},
  "capabilityManifest": {},
  "validationSummary": {},
  "testSummary": {},
  "assets": []
}
```

Deterministic array rules:

- preserve Editor Graph, Runtime IR, schema, validation, and test array order because it is content;
- sort distinct Node Definition entries by `type`;
- sort each definition's capability codes lexicographically;
- sort the capability union lexicographically;
- after Asset integration, sort assets by normalized alias, then `assetHash`.

Excluded from `contentHash`: database IDs, Project ID, source draft version/hash, creator, timestamps,
Submission/Product identifiers, and scan/review state. Those fields establish identity or audit
context but do not change the frozen technical content.

Before creating the revision, the Backend recomputes the five-field Project Draft hash and requires
it to equal stored `ProjectDraft.contentHash`. Revision canonical content is limited to 2 MiB
(`2_097_152` UTF-8 bytes); use a named constant rather than a business-logic literal.

## 6. Creation service contract

The internal application contract is intentionally small:

```ts
createFromDraft({
  projectId,
  sourceDraftVersion,
  createdById,
  access,
}): Promise<{
  id: string;
  projectId: string;
  sourceDraftVersion: number;
  sourceDraftContentHash: string;
  contentHash: string;
}>;
```

`access` is the typed result of the existing Project/Publisher access policy. The service does not
accept workflow JSON, manifests, hashes, timestamps, Publisher IDs, or role names from its
caller.

The future Submission service remains responsible for resolving Publisher eligibility and invoking
this method within its transaction. The Workflow Revision repository still applies the typed Project
ownership predicate so a caller cannot accidentally snapshot a cross-owner Project.

## 7. Transaction and concurrency

The future Submission transaction should use this order:

1. Resolve active Publisher submission access.
2. Lock the accessible Project row; reject missing/cross-owner or archived Projects.
3. Lock the non-deleted Project Draft row and require `version === sourceDraftVersion`.
4. Recompute and verify the source draft hash; require non-null object `runtimeIr`.
5. Extract distinct node types and load current definitions/capabilities in one set-based query.
6. Build the frozen manifests and deterministic revision hash without network calls.
7. Insert the Workflow Revision.
8. Snapshot the locked existing Product draft, then create Submission and the outbox event in the
   same transaction. Submission never creates or edits Product.

Lock Project before Project Draft everywhere to keep lock ordering stable. Database uniqueness on
`(projectId, sourceDraftVersion)` is the durable concurrency boundary; a Redis lock is not required.

Submission-level idempotency owns replay after a committed response is lost. Workflow Revision does
not silently reuse an existing row to create a second Submission. A uniqueness collision not resolved
by the enclosing idempotency contract is a domain conflict/invariant failure.

Do not perform validation queue publication, storage access, Redis work, or other network I/O while
the database transaction is open. Publish the durable outbox only after commit through the later
outbox relay.

## 8. Eligibility and validation boundary

Synchronous revision creation verifies only conditions required to produce a coherent snapshot:

- active accessible Project and exact current draft version;
- valid stored draft hash and object-shaped content;
- non-null object `runtimeIr`;
- all distinct graph node types resolve to active Node Definitions;
- derived manifests and revision content satisfy size/shape/hash constraints;
- no unsupported asset references during the asset-free phase.

It does not trust Studio's `validationSummary` or `testSummary`. It also does not silently compile,
repair, or normalize publisher workflow semantics.

After commit, the Submission pipeline runs the reusable cloud Workflow Validation service against
the immutable `editorGraph`, persists the Backend result outside Workflow Revision, and continues to
advanced Runtime IR, asset, capability, and policy scans only when allowed. Current Basic Workflow
Validation does not prove Runtime IR consistency; production release must remain gated until the
required advanced checks exist.

## 9. Authorization and reads

- There is no Workflow Revision route permission because this phase adds no public controller.
- Only an active, non-deleted Publisher account with owned submission access can cause creation.
- Admin unrestricted Project access does not by itself grant authority to submit Publisher content.
- Missing and cross-owner Projects are indistinguishable to the external Submission API.
- Internal read methods select only fields required by Submission scanning, review, package building,
  or release creation.
- Web reads revisions through future Submission detail contracts; do not expose a general revision
  listing endpoint merely because the table exists.

## 10. Stable failure semantics

These errors are surfaced by the future Submission API; Workflow Revision has no standalone HTTP
adapter:

| Condition                                                  | Recommended result                                                             |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Missing or cross-owner Project/draft                       | `404 Error.ProjectNotFound`                                                    |
| Archived Project                                           | `409 Error.ProjectArchived`                                                    |
| Requested version is no longer current                     | `409 Error.WorkflowRevisionDraftVersionConflict`, safe `currentVersion` detail |
| Missing/null Runtime IR                                    | `409 Error.WorkflowRevisionRuntimeIrRequired`                                  |
| Missing/disabled node type or changed invalid definition   | `409 Error.WorkflowRevisionNodeDefinitionUnavailable`                          |
| Stored draft hash mismatch                                 | `409 Error.WorkflowRevisionSourceIntegrityFailed`                              |
| Revision canonical content exceeds 2 MiB                   | `413 Error.WorkflowRevisionPayloadTooLarge`                                    |
| Asset reference before Asset integration                   | `409 Error.WorkflowRevisionAssetUnsupported`                                   |
| Concurrent duplicate not handled by Submission idempotency | `409 Error.WorkflowRevisionAlreadyExists`                                      |

Do not include graph, Runtime IR, bindings, credentials, test inputs, asset URLs, or private payloads
in exception messages or safe error details.

## 11. Logging, audit, and operations

- Do not add routine success logs for revision insertion.
- Central request/error logging covers expected failures without serializing workflow content.
- Log unexpected integrity failures with identifiers and hashes only, never the content itself.
- The future Submission mutation owns its durable audit entry and outbox event; Workflow Revision
  must not create a second non-atomic audit path.
- A revision has no retention/delete operation in the MVP. Administrative SQL repair is an
  exceptional operational process, not an application feature.

## 12. Implementation map

Expected Backend files after implementation:

| File                            | Responsibility                                                           |
| ------------------------------- | ------------------------------------------------------------------------ |
| `workflow-revision.module.ts`   | Private providers exported only to Submission                            |
| `workflow-revision.service.ts`  | Snapshot orchestration and invariants                                    |
| `workflow-revision.repo.ts`     | Transaction-aware locks, batched reads, numbering, insert, trusted reads |
| `workflow-revision.model.ts`    | Internal input/output and frozen JSON schemas                            |
| `workflow-revision.error.ts`    | Stable domain exceptions                                                 |
| `workflow-revision.util.ts`     | Canonicalization, manifest construction, size and hash calculation       |
| `workflow-revision.constant.ts` | Hash version, size limit, safe stable constants                          |
| `test/*.spec.ts`                | Hash, manifest, service, repository, concurrency, and boundary tests     |

No controller or DTO file is expected in this phase. Reuse or extract the existing canonical JSON
helper only if the resulting shared contract is explicit and both Draft and Revision tests protect
it; avoid speculative generic utilities.

## 13. Acceptance criteria for future implementation

- [ ] No direct public mutation endpoint exists for Workflow Revision.
- [ ] Exact current draft version is locked and cannot race with a Studio save.
- [ ] Missing, archived, cross-owner, stale-version, and missing-Runtime-IR cases fail stably.
- [ ] One draft version creates at most one revision under concurrent requests.
- [ ] Full definitions for distinct graph node types are loaded in one set-based query and frozen.
- [ ] Missing/disabled definitions fail creation; their catalog publication hashes are preserved.
- [ ] Capability requirements are Backend-derived, deduplicated, and sorted.
- [ ] Source draft and revision hashes are deterministic across JSON object key order.
- [ ] Hash changes when any included workflow, manifest, summary, or asset-hash content changes.
- [ ] Audit/identity-only fields do not change `contentHash`.
- [ ] JSON shapes, positive draft versions, lowercase hashes, and source-version uniqueness exist in
      PostgreSQL.
- [ ] No update, soft-delete, hard-delete, or revision-content replacement path exists.
- [ ] Revision and Submission commit together or roll back together.
- [ ] Workflow payloads and secrets never appear in logs or safe error details.
- [ ] Asset references are rejected until immutable Asset snapshot support is implemented.
- [ ] SQL migration is created only; deployment remains a user action under repository policy.

## 14. Deferred

- Public Submission and Submission detail APIs.
- Submission idempotency key and durable replay behavior.
- Asset/Storage modules and `WorkflowRevisionAsset` persistence.
- Advanced binding, conditional, loop, retry, error-flow, and Runtime IR consistency validation.
- Automated policy scan, scan findings, review, packaging, encryption, and Product Release.
- Node Catalog version tables; full definition snapshots are the approved MVP alternative.
- Revision retention or administrative repair tooling.
