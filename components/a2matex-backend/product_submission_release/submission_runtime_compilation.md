# Stage 4 — Submission Runtime IR Compilation

## Overview

The primary change in Stage 4 is integrating the existing `WorkflowCompilerService` into the
existing Submission scan consumer. Stage 4 does not create another compiler or another background
job. The HTTP request freezes publisher-authored source and the exact Node Definitions, commits the
Submission, and publishes the existing scan message. The consumer validates that frozen source
first through `WorkflowCompilerService.validateWorkflow()`. Only its successful parsed result is
passed to `compileValidatedWorkflow()`. Successful compilation atomically stores the authoritative
Runtime IR, final Workflow Revision hash, scan result, and review-ready state. Validation failure
stores `FAILED_VALIDATION` and stops before compilation. Deterministic compilation failure stores
`FAILED_COMPILATION`. Both outcomes retain Admin-visible findings and
leave Runtime IR null.

```mermaid
flowchart LR
    P[Publisher creates Submission] --> H[HTTP transaction]
    H --> F[Freeze draft source and Node Definitions]
    F --> S[Create pending WorkflowRevision and Submission]
    S --> K[Publish existing scan message after commit]
    K --> C[Existing Submission scan consumer]
    C --> V[Run workflow validation with frozen definitions]
    V --> G{Validation valid?}
    G -- No --> B[FAILED_VALIDATION<br/>Runtime IR null, READY_FOR_REVIEW]
    G -- Yes --> W[Compile validated frozen workflow]
    W --> X{Compilation succeeds?}
    X -- No: compiler diagnostics --> F[FAILED_COMPILATION<br/>Runtime IR null, READY_FOR_REVIEW]
    X -- Yes --> A[Atomically save Runtime IR, final hash, PASSED result, READY_FOR_REVIEW]
    C -- Infrastructure failure --> R[Keep scan ERROR for existing Admin retry]
    W -- Technical exception --> R
    A --> D[Product Release later packages stored Runtime IR]
    B --> E[Admin sees findings and provides Publisher feedback]
    F --> E
```

This stage owns the consumer-to-compiler integration and Workflow Revision finalization. Admin approval and
Product Release building remain owned by their existing/later modules. It does not add another
queue, outbox, build record, or public compilation endpoint.

## 1. Existing implementation

The current Backend already provides:

- Submission creation with idempotency, Product and Project locking, snapshots, one open
  Submission rule, and post-commit Kafka publication.
- `WorkflowRevisionService.createFromDraft()`, called inside the Submission transaction.
- A Submission scan consumer with attempt checks, `QUEUED -> RUNNING -> PASSED|FAILED_VALIDATION|FAILED_COMPILATION`,
  technical `ERROR`, Admin retry, bounded findings, and Admin-only full scan details.
- `WorkflowValidationService` and the reusable `WorkflowCompilerService.compileWorkflow()`.
- A frozen Node Definition and capability manifest on `WorkflowRevision`.

The current behavior that Stage 4 replaces is:

1. `WorkflowRevisionService` requires non-null `ProjectDraft.runtimeIr`.
2. Revision creation copies that draft Runtime IR immediately.
3. Revision creation calculates the final `contentHash` before the scan runs.
4. The scanner validates the Editor Graph against the live Node Catalog.
5. The scanner does not validate or generate Runtime IR, but may still promote the Submission to
   `READY_FOR_REVIEW`.

Consequently, Studio currently has to copy preview Runtime IR into the cloud draft. That is a
temporary bridge and must no longer be required after this stage.

## 2. Scope

### Included

- Freeze a pending Workflow Revision without Runtime IR or final content hash.
- Inject and call the existing `WorkflowCompilerService` from the existing Submission scan
  consumer after workflow validation succeeds.
- Use the frozen Node Definition manifest, never mutable catalog rows, during the scan.
- Atomically finalize the Workflow Revision and Submission scan outcome.
- Preserve existing Submission message, attempt, retry, visibility, and review behavior.
- Update response contracts to allow a pending or failed revision to have null compiled fields.

### Excluded

- Admin review decision APIs.
- Product Release package building.
- Product Offer, Order, payment, or entitlement behavior.
- A second Kafka topic, outbox, build table, or scheduled job.
- Compiler versioning, Runtime version redesign, assets, or capability-manifest redesign.
- Removing legacy `ProjectDraft.runtimeIr` columns in this stage.

## 3. Data model changes

The existing migration makes the compiled Workflow Revision fields nullable. Prepare a separate
forward migration for the explicit scan-failure statuses.

```prisma
model WorkflowRevision {
  // unchanged frozen source fields
  editorGraph            Json
  inputSchema            Json
  nodeDefinitionManifest Json
  capabilityManifest     Json
  validationSummary      Json

  // pending until a successful scan compilation
  runtimeIr              Json?
  contentHash            String? @db.VarChar(64)
}
```

The scan-status migration replaces `SubmissionScanStatus` with `QUEUED`, `RUNNING`, `PASSED`,
`ERROR`, `FAILED_VALIDATION`, and `FAILED_COMPILATION`. `Submission.scanStatus` owns scan lifecycle, including
pending compilation.

Create the enum migration with the Backend README command.

```bash
npx prisma migrate dev --create-only --config prisma7.config.ts
```

Its SQL recreates the enum with those six values and maps existing `FAILED` scans to
`FAILED_VALIDATION` while casting `Submission.scanStatus` to the new type. Other scan statuses
and stored scan results are preserved. Migration deployment remains a manual operator action
after review.

## 4. Submission HTTP transaction

The public API contract and create response remain unchanged. Creation still returns only:

```json
{ "id": "submission-uuid" }
```

Inside the existing transaction:

1. Resolve active Publisher access and lock the owned Product using the existing lock order.
2. Validate Product status and ordered Product media, then build the existing snapshots.
3. Lock the Product's Project and exact `ProjectDraft`.
4. Require `draft.version == sourceDraftVersion`.
5. Recalculate and verify the Project Draft source hash. It covers `editorGraph + inputSchema`.
6. Validate the asset-free MVP restriction.
7. Extract distinct node types and load all active Node Definitions once.
8. Fail creation if any referenced definition is unavailable.
9. Freeze `editorGraph`, `inputSchema`, Node Definition manifest, capability manifest,
   `validationSummary`, source draft version, and source draft hash.
10. Create `WorkflowRevision` with `runtimeIr = null` and `contentHash = null`.
11. Create the Submission in `PENDING_SCAN / QUEUED`.
12. Publish the existing scan message only after commit.

The handler must not compile, perform Runtime work, or contact Kafka while database locks are held.
It must no longer reject a draft because `ProjectDraft.runtimeIr` is null.

## 5. Frozen Node Definition contract

The consumer must compile with `WorkflowRevision.nodeDefinitionManifest`. It must not query the
current Node Catalog, because catalog activation or edits after Submission must not change the
result of a retry.

At the consumer boundary:

- Strictly parse the manifest envelope and every definition.
- Convert stored `capabilityCodes` to compiler `capabilities` without changing order or content.
- Reject duplicate or missing node types as a technical frozen-data integrity failure.
- Require every Editor Graph node type to have exactly one frozen definition.

Malformed system-owned manifests are technical failures and follow the existing scan `ERROR` retry
path. Publisher-authored graph or binding problems are deterministic scan findings.

## 6. Consumer pipeline

The existing Submission scan handler and message remain the entry point. This section is the core
of Stage 4: validation gates a call from the consumer into the existing compiler service.

For the current attempt:

1. Load the Submission and frozen Workflow Revision.
2. Apply existing stale, future, terminal, and claim checks.
3. Parse frozen source and manifests.
4. Run source-integrity, snapshot, and asset checks before workflow compilation.
5. Call `WorkflowCompilerService.validateWorkflow()` with frozen source and parsed frozen Node
   Definitions. It reuses the existing pure Editor Graph, topology, catalog, binding, and Input
   Schema validation rules.
6. If a content check or workflow validation returns errors, return `FAILED_VALIDATION` with the
   findings immediately. The common persistence path saves the outcome and leaves compiled fields
   null. IR generation is skipped.
7. When validation succeeds, pass its parsed result to the existing compiler service.

```ts
const validated = workflowCompilerService.validateWorkflow(
  revision.editorGraph,
  revision.inputSchema,
  frozenDefinitions,
);
// Validation failure returns FAILED_VALIDATION through the scanner outcome.
if (validated.success) {
  const compiled = workflowCompilerService.compileValidatedWorkflow(validated);
}
```

8. The compilation step consumes validated inputs directly. It performs IR construction and strict
   Runtime IR output checks. The existing `compileWorkflow()` preview entry point reuses the same
   two steps and keeps its public response contract.
9. Deterministic IR contract, size, capability-consistency, or final revision-size errors return
   `FAILED_COMPILATION` with findings. Unexpected exceptions propagate to the existing technical
   `ERROR` recovery path.
10. On success, calculate the final Workflow Revision hash and atomically persist Runtime IR, hash,
    `PASSED`, and the scan result.

The scanner returns the final `scanStatus` with `scanResult` and optional compiled output. The
scan service persists that status directly. The compiler carries its ordinary success result and
parsed inputs between calls, while the scanner owns Submission lifecycle decisions.

The compiler must never read `ProjectDraft.runtimeIr`. Preview Runtime IR and Studio validation are
not compilation authorities.

The compiler call is therefore ordered strictly as:

```text
load frozen revision
  -> validate frozen source
  -> stop with FAILED_VALIDATION when invalid
  -> compile with WorkflowCompilerService when valid
  -> stop with FAILED_COMPILATION on compiler diagnostics
  -> atomically finalize Runtime IR and PASSED scan result
```

## 7. Atomic persistence

Extend the existing transactional scan outcome write. For a successful scan, one database
transaction must:

1. Conditionally write `WorkflowRevision.runtimeIr` and `contentHash` only when both are null.
2. Write `Submission.scanStatus = PASSED`, scanner version, and bounded scan result for the same
   `scanAttempt`.
3. Move `Submission.status` from `PENDING_SCAN` to `READY_FOR_REVIEW` for that same attempt.

For deterministic invalid content, the same transaction must:

1. Leave both Workflow Revision compiled fields null.
2. Write `FAILED_VALIDATION` or `FAILED_COMPILATION` according to the failing phase and save findings.
3. Move the Submission to `READY_FOR_REVIEW` so Admin can inspect findings and reject with useful
   feedback.

If the attempt was superseded, no revision or Submission result may be written. Do not leave a
successful Runtime IR committed separately from its scan result.

## 8. Idempotency and retries

- `PASSED`, `FAILED_VALIDATION`, and `FAILED_COMPILATION` are terminal.
  Redelivery returns the existing `ALREADY_DONE` outcome.
- A stale attempt performs no writes.
- A crash before the final transaction commits leaves both the revision and scan unfinished; Kafka
  redelivery safely repeats validation and compilation from immutable inputs.
- A successful finalization uses a conditional null-to-value update. It never overwrites an
  existing authoritative Runtime IR.
- Concurrent delivery may perform duplicate in-memory computation, but only one identical
  finalization may win; no additional builder record or lock is introduced.
- A technical exception leaves the revision pending and uses the current `scanStatus = ERROR`
  recovery path.

## 9. Failure classification and visibility

### Deterministic business failures

Examples:

- Invalid Editor Graph, topology, port, binding, or input schema.
- Unsupported asset usage.
- Runtime IR exceeds its contract limit.
- Compiler returns publisher-content diagnostics.

`FAILED_VALIDATION` covers invalid source, graph, binding, Input Schema, snapshot, or asset
checks. `FAILED_COMPILATION` covers deterministic errors after validation succeeds. Both are
completed business scans and return Kafka `SUCCESS` after persistence.

Store bounded findings in the existing `Submission.scanResult`. Full findings remain visible only
through the Admin detail projection. The Publisher sees review state and later Admin-authored
feedback, not private server scan details.

### Technical failure (`ERROR`)

Examples:

- Database or Kafka infrastructure failure.
- Malformed frozen system-owned Node Definition manifest.
- Failure to atomically persist the final outcome.

These failures are retryable through the existing Admin retry operation and must not create a
business scan result.

The existing result shape is retained. `summary.totalRules` counts evaluated scanner checks and
`summary.passed` counts checks without error findings. A complete scan evaluates five checks
in execution order. They cover frozen-source integrity, snapshot integrity, asset restrictions,
frozen workflow validation, and compilation/finalization. Early returns report only evaluated
checks. Findings and warning counts include earlier successful stages, and truncation propagates
across both phases.
Scanner version `0.4.0` identifies these phase and summary semantics.

## 10. API contract adjustments

No new endpoint is added. Existing Submission endpoints remain protected by their current access
rules.

While a scan is pending or failed, detail serialization must allow:

```ts
workflowRevision.runtimeIr: object | null;
workflowRevision.contentHash: string | null;
```

Owner projections must continue omitting `scanResult`, `scannerVersion`, and internal attempt
fields. Admin projections retain them. List endpoints must not preload Runtime IR.

## 11. Implementation mapping

| Area | Required change |
|---|---|
| Prisma migration and schema | Make `WorkflowRevision.runtimeIr` and `contentHash` nullable |
| `workflow-revision.model.ts` | Pending snapshot/output types accept null compiled fields |
| `workflow-revision.util.ts` | Split source snapshot preparation from final hash calculation |
| `workflow-revision.service.ts` | Remove draft Runtime IR requirement; create pending revision |
| `workflow-revision.repo.ts` | Insert null compiled fields; add conditional finalization method |
| `submission.model.ts` | Detail response accepts null revision Runtime IR/hash |
| `submission.repo.ts` | Load frozen manifests; atomically finalize revision and scan outcome |
| `submission-scanner.service.ts` | Validate frozen input, then inject and call `WorkflowCompilerService`; never duplicate compiler logic |
| `submission-scan.service.ts` | Carry compiled Runtime IR/final hash into transactional persistence |
| Module wiring | Import/export `WorkflowCompilerModule` where the scan consumer needs it |

Do not duplicate compiler logic in Submission code.

## 12. Verification

Required automated coverage:

- Submission creation succeeds when `ProjectDraft.runtimeIr` is null.
- Revision creation freezes the exact requested draft version and definitions.
- Missing frozen node definition prevents Submission creation.
- Invalid workflow validation returns `FAILED_VALIDATION`, skips `compileValidatedWorkflow()`,
  and leaves Runtime IR null.
- Valid workflows run validation once against frozen inputs, then pass parsed inputs directly to
  `compileValidatedWorkflow()` once.
- Scan compilation succeeds and stores Runtime IR plus final hash.
- Compiler diagnostics produce `FAILED_COMPILATION`, null compiled fields, and Admin findings.
- Both explicit failure statuses allow Admin request-changes/rejection and block approval.
- Summary counts reflect early returns and retain warnings from earlier stages.
- Technical compiler/manifest exceptions produce `ERROR` without a business result.
- A stale attempt cannot finalize a revision.
- Redelivery after a pre-commit crash safely recompiles.
- Terminal redelivery returns `ALREADY_DONE` without rewriting Runtime IR.
- Publisher detail omits private scan findings; Admin detail includes them.
- Existing create idempotency and one-open-Submission behavior remain unchanged.

Run focused Workflow Revision, Submission service, scanner, consumer-handler, compiler, build, lint,
and migration validation checks. Do not apply the migration.

## 13. Completion criteria

Stage 4 is complete when:

- Studio no longer needs to save preview Runtime IR before Submission.
- Submission creation commits a pending immutable Workflow Revision and publishes the existing scan
  request.
- The existing scan consumer generates IR only after frozen-source validation succeeds, using
  the shared compiler service for both phases.
- The compiler receives only frozen source and frozen definitions; Submission code contains no
  duplicate Graph-to-IR transformation.
- `READY_FOR_REVIEW` cannot contain a passed scan with null Runtime IR or final hash.
- Invalid publisher content reaches Admin review with private findings and no authoritative Runtime
  IR.
- Retries cannot overwrite a finalized revision or apply a stale attempt.
- No new queue architecture or Product Release behavior was introduced.

