# Feature and Use-Case Catalog

This catalog is intentionally product-oriented. A BA can later convert it into a
formal SRS with complete preconditions, postconditions, acceptance criteria, and
traceability.

## 1. Feature inventory

### 1.1 Developer Studio

| ID | Feature | Priority |
|---|---|---|
| F-STU-01 | Create/open/save local project | UAT must |
| F-STU-02 | Visual node/edge canvas with stable node IDs | UAT must |
| F-STU-03 | Node Inspector: config, input, code, output, test, result | UAT must |
| F-STU-04 | System Node Registry and version pin | UAT must |
| F-STU-05 | Custom TypeScript Node with Monaco, typings, format, lint | UAT must |
| F-STU-06 | Custom Integration through Platform SDK | UAT must |
| F-STU-07 | Conditional edge and branch mode | UAT must |
| F-STU-08 | Merge and activation policy | UAT must |
| F-STU-09 | Controlled dynamic Loop Controller | UAT must |
| F-STU-10 | Run complete workflow | UAT must |
| F-STU-11 | Run one node with pinned input | UAT must |
| F-STU-12 | Per-node input/output/error inspection | UAT must |
| F-STU-13 | Local autosave and revision journal | UAT must |
| F-STU-14 | Register Project and show quota | UAT must |
| F-STU-15 | Freeze and submit immutable Revision | UAT must |
| F-STU-16 | Local Private Node Library | UAT should |

### 1.2 Platform, Build, and Review

| ID | Feature | Priority |
|---|---|---|
| F-PLT-01 | Publisher and Registered Project management | UAT must |
| F-PLT-02 | Plan limits for slots, submissions, builds, storage | UAT must |
| F-PLT-03 | Snapshot upload and integrity verification | UAT must |
| F-PLT-04 | Immutable Revision and Submission state | UAT must |
| F-BLD-01 | Structural graph validation | UAT must |
| F-BLD-02 | TypeScript/source/security validation | UAT must |
| F-BLD-03 | Capability/domain extraction | UAT must |
| F-BLD-04 | Workflow IR compilation | UAT must |
| F-BLD-05 | Candidate `.apkg` build | UAT must |
| F-BLD-06 | Sandbox test with mock capability/credential | UAT must |
| F-REV-01 | Graph, source, diff, risk, permission review | UAT must |
| F-REV-02 | Approve/reject/request changes/escalate | UAT must |
| F-SIG-01 | Exact-hash Artifact signing | UAT must |
| F-OPS-01 | Audit trail | UAT must |
| F-OPS-02 | Product/version/account/device revocation | UAT must |

### 1.3 Web Marketplace

| ID | Feature | Priority |
|---|---|---|
| F-MKT-01 | Product listing and Product Version metadata | UAT must |
| F-MKT-02 | Public catalog and Product detail | UAT should |
| F-MKT-03 | Declared permission/domain/credential disclosure | UAT must |
| F-MKT-04 | Customer acquisition or Admin grant | UAT must |
| F-MKT-05 | Customer web library and entitlement status | UAT must |
| F-MKT-06 | Publisher Submission/Version status | UAT must |
| F-MKT-07 | Basic execution analytics | UAT should |
| F-MKT-08 | Real payment, refund, payout | After UAT |

### 1.4 Android Launcher

| ID | Feature | Priority |
|---|---|---|
| F-LAU-01 | Login and Customer Library sync | UAT must |
| F-LAU-02 | Resumable download | UAT must |
| F-LAU-03 | Hash, signature, compatibility, revocation verification | UAT must |
| F-LAU-04 | Atomic install in private storage | UAT must |
| F-LAU-05 | Permission disclosure and product-level grants | UAT must |
| F-LAU-06 | Android permission request at relevant action | UAT must |
| F-LAU-07 | Customer credential connection and binding | UAT must |
| F-LAU-08 | Run/stop and Customer Monitor | UAT must |
| F-LAU-09 | Signed License Lease and bounded offline run | UAT must |
| F-LAU-10 | Product update | UAT must |
| F-LAU-11 | Rollback to retained version | UAT should |

### 1.5 Runtime Agent

| ID | Feature | Priority |
|---|---|---|
| F-RUN-01 | Package and Lease verification | UAT must |
| F-RUN-02 | Workflow IR loader | UAT must |
| F-RUN-03 | Graph scheduler | UAT must |
| F-RUN-04 | Conditional, merge, activation, and loop semantics | UAT must |
| F-RUN-05 | Restricted JavaScript sandbox | UAT must |
| F-RUN-06 | Versioned System Node handlers | UAT must |
| F-RUN-07 | Capability Broker | UAT must |
| F-RUN-08 | Credential reference resolution | UAT must |
| F-RUN-09 | Timeout, quota, retry, and cancellation | UAT must |
| F-RUN-10 | Execution events and sanitized logs | UAT must |

## 2. Use-case inventory

| ID | Use case | Primary actor | Product |
|---|---|---|---|
| UC-001 | Create a local project | Creator | Studio |
| UC-002 | Design a workflow graph | Creator | Studio |
| UC-003 | Create a Custom TypeScript Node | Creator | Studio |
| UC-004 | Configure a Custom Integration Node | Creator | Studio |
| UC-005 | Create a conditional branch | Creator | Studio |
| UC-006 | Merge multiple upstream branches | Creator | Studio |
| UC-007 | Create a controlled loop | Creator | Studio |
| UC-008 | Run a complete workflow | Creator | Studio/Runtime |
| UC-009 | Run one node with pinned input | Creator | Studio/Runtime |
| UC-010 | Inspect node input/output/error | Creator | Studio |
| UC-011 | Register a project | Publisher | Studio/Web |
| UC-012 | Freeze and submit a Revision | Publisher | Studio/Platform |
| UC-013 | Validate and build a candidate | Build Service | Platform |
| UC-014 | Review a Submission | Security Reviewer | Web Admin |
| UC-015 | Request Publisher changes | Reviewer | Web Admin |
| UC-016 | Approve and sign an Artifact | Reviewer/Admin | Platform |
| UC-017 | Publish a Product Version | Publisher/Admin | Web |
| UC-018 | Browse Product catalog | Guest/Customer | Marketplace |
| UC-019 | Acquire a Product | Customer/Admin | Marketplace |
| UC-020 | Synchronize Customer Library | Customer | Launcher |
| UC-021 | Download and install `.apkg` | Customer | Launcher |
| UC-022 | Grant Product permission | Customer | Launcher |
| UC-023 | Connect Customer credential | Customer | Launcher |
| UC-024 | Acquire or refresh License Lease | Launcher | Platform |
| UC-025 | Run Product online | Customer | Launcher/Runtime |
| UC-026 | Run Product offline | Customer | Launcher/Runtime |
| UC-027 | Monitor an Execution | Customer | Launcher |
| UC-028 | Stop an Execution | Customer | Launcher/Runtime |
| UC-029 | Revoke permission during Execution | Customer | Launcher/Runtime |
| UC-030 | Disconnect or rebind Credential | Customer | Launcher |
| UC-031 | Update a Product Version | Customer | Launcher |
| UC-032 | Roll back a Product Version | Customer/Support | Launcher |
| UC-033 | View basic analytics | Publisher/Admin | Web |
| UC-034 | Revoke Product or Version | Platform Admin | Web/Platform |
| UC-035 | Investigate Audit Trail | Admin/Support | Web |
| UC-036 | Save and reuse a local private node | Creator | Studio |

## 3. Critical use cases

### UC-012 - Freeze and submit a Revision

**Preconditions**

- Project is registered.
- Publisher is authenticated and authorized.
- Project, submission, build, and storage quota are available.

**Main flow**

1. Publisher selects Submit.
2. Studio runs preflight validation.
3. Studio freezes graph, source, metadata, and asset hashes.
4. Studio compresses and hashes the snapshot.
5. Snapshot is uploaded with resume/retry support.
6. Platform verifies hash, size, file count, expanded size, and ratio.
7. Platform creates immutable Revision and Submission.
8. Automated validation starts.

**Important exceptions**

- Quota exceeded: reject before starting an unnecessary build.
- Hash mismatch: reject and audit.
- Unsafe archive: reject and quarantine.
- Interrupted upload: resume or retry idempotently.

**Result**

The Revision cannot change. The local Draft may continue to evolve independently.

### UC-013 - Validate and build a candidate

**Main flow**

1. Load the immutable snapshot.
2. Run structural validation.
3. Run compiler, lint, AST, import, secret, and capability checks.
4. Resolve and lock System Node, SDK, and utility versions.
5. Compile Workflow IR and custom script modules.
6. Generate manifest, permission file, node lock, and hashes.
7. Create candidate `.apkg`.
8. Run sandbox tests.
9. Save reports and candidate hash.

**Result**

The candidate is reviewable but cannot be distributed as a production release.

### UC-014 - Review a Submission

**Preconditions**

- Candidate Artifact and automated reports are available.

**Main flow**

1. Reviewer opens the Submission.
2. Web Admin displays graph and previous-version diff.
3. Reviewer checks source and source diff.
4. Reviewer checks domains, capabilities, credential slots, and side effects.
5. Reviewer checks sandbox and resource warnings.
6. Reviewer records findings and reason.
7. Reviewer approves, rejects, requests changes, or escalates.

**Result**

Decision is immutable and tied to one candidate hash.

### UC-016 - Approve and sign an Artifact

1. Approval creates a signing request containing the approved candidate reference
   and hash.
2. Signing Service fetches the immutable candidate.
3. Signing Service recalculates and compares the hash.
4. If hashes match, it signs the Artifact.
5. Signed Artifact, signer/key identity, and audit record are stored.
6. Product Version becomes eligible for publication.

Hash mismatch must fail closed and create a security event.

### UC-021 - Download and install `.apkg`

1. Customer selects Install.
2. Launcher downloads or resumes the Artifact.
3. Launcher checks size and expected hash.
4. Launcher verifies platform signature and signer status.
5. Launcher verifies Product ID, version, Runtime range, and revocation.
6. Launcher parses the manifest and shows capability summary.
7. Launcher moves the verified package atomically into private installed storage.

Failure never destroys the current active version.

### UC-022 - Grant Product permission

1. Launcher shows capability and purpose.
2. Customer allows or denies the capability.
3. If Android permission is needed, Launcher requests it at that action.
4. Product grant is stored.
5. Runtime applies the grant to future calls.

An Android grant alone is insufficient; a Product grant alone is also
insufficient where Android permission is required.

### UC-023 - Connect Customer credential

1. Launcher shows the required credential slot and purpose.
2. Customer completes OAuth/provider authorization.
3. Vault stores token material.
4. Launcher binds only the credential reference to the Product slot.
5. Runtime checks the binding when the node runs.

Cancelled OAuth leaves the slot unbound. Missing provider scopes require
reconnection; the workflow never receives a silently under-scoped credential.

### UC-024 - Acquire or refresh License Lease

1. Launcher sends Product, version, device, session, and nonce.
2. Server validates account, Entitlement, Product Version, device, and policy.
3. Server signs a Lease containing refresh and offline boundaries.
4. Launcher verifies and stores the Lease securely.
5. After `refreshAfter`, Launcher refreshes in the background when online.

If the server is unavailable while an existing Lease remains valid, execution
continues according to the signed Lease.

### UC-025 - Run Product online

1. Customer selects Run.
2. Launcher verifies installed package, current Lease, required grants, and
   credential bindings.
3. Runtime loads Workflow IR and creates an Execution Context.
4. Scheduler executes ready nodes.
5. Capability Broker handles approved network/hardware operations.
6. Customer Monitor receives progress events.
7. Runtime records final state and permitted logs.

### UC-026 - Run Product offline

1. Runtime verifies package and Lease locally.
2. Runtime checks `offlineUntil`, device, Product, and version.
3. Cached Product grants and offline-usable credential state are checked.
4. Workflow runs.
5. Network nodes fail or retry according to workflow policy.
6. Local permitted logs are retained for display or later aggregate sync.

An expired Lease requires online refresh. Moving the device clock backward also
requires online trust refresh.

### UC-028 - Stop an Execution

1. Customer selects Stop.
2. Launcher sends cancellation to Runtime.
3. Scheduler stops starting new nodes.
4. Capability Broker cancels pending operations where possible.
5. Running handler receives cancellation or reaches timeout.
6. Execution ends as `CANCELLED`.

No new downstream side effect may begin after cancellation is acknowledged.

### UC-029 - Revoke permission during Execution

1. Customer revokes a Product grant.
2. Grant changes to `DENIED`.
3. Runtime receives a permission-change event.
4. Broker cancels not-yet-started capability work.
5. Later calls fail with `PERMISSION_REVOKED`.
6. Workflow uses its error edge or stops.

### UC-031 - Update a Product Version

1. Launcher detects a published update allowed by Entitlement.
2. New Artifact downloads and verifies separately.
3. Launcher calculates capability/domain/credential delta.
4. Increased scope requires new Customer disclosure and consent.
5. New version activates atomically.
6. Previous version remains according to rollback retention.

Verification failure leaves the old version active.

### UC-034 - Revoke a Product or Version

1. Admin selects target, severity, and reason.
2. Platform updates revocation state and audit.
3. New acquisition, install, or Lease issuance is blocked as policy requires.
4. Launcher receives revocation at the next trusted online check.
5. Runtime blocks execution according to current trusted state and signed offline
   policy.

Revocation preserves required metadata, hash, and audit. It is not equivalent to
deleting history.

