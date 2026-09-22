# UAT Scope and Delivery Plan

## 1. UAT scope lock

### 1.1 Required

#### Developer Studio

- Local project.
- Visual node/edge canvas.
- System Nodes.
- Custom TypeScript Node.
- Custom Integration through Platform SDK.
- Conditional edges.
- Merge and activation policy.
- Controlled dynamic loops.
- Formatter, linter, and type diagnostics.
- Run complete workflow.
- Run one node with pinned input.
- Inspect per-node input, output, error, and duration.
- Submit immutable Revision.

#### Platform and Admin

- Publisher and Registered Project management.
- Registered Project and Publish Request limits.
- Snapshot upload.
- Candidate `.apkg` build.
- Basic automated validation.
- Admin review of graph, source, diff, permission, domain, and sandbox result.
- Approve, reject, request changes, and escalate.
- Artifact signing.
- Product listing.
- Entitlement and signed License Lease.
- Audit.

#### Web Marketplace

- Product detail and permission/domain disclosure.
- Customer acquisition through mock purchase or Admin grant.
- Customer Library.
- Publisher Submission and Version status.
- Protected Admin/Reviewer operations.

#### Android Launcher

- Login and Library.
- Download, verify, and install `.apkg`.
- Shared Runtime integration.
- Android and Product-level permission.
- Credential connection.
- Customer Monitor.
- Stop execution.
- Bounded offline License Lease.
- Update; basic rollback if time permits.

#### Runtime Agent

- Package and Lease verification.
- Workflow IR loader.
- Scheduler with conditional, merge, activation, and loop.
- Restricted TypeScript execution.
- Capability Broker.
- HTTP plus selected Camera/Bluetooth capability.
- Quota, timeout, retry, and cancellation.
- Execution events and sanitized logs.

### 1.2 After UAT

- Team collaboration.
- AI suggestions.
- Public Node Marketplace.
- Cloud Private Node Library.
- Platform-managed Publisher secret proxy.
- Python, Go, and multi-language runtime.
- Native Custom Nodes.
- Full Security Engine.
- Complete Play Integrity/device attestation.
- Advanced analytics.
- Cloud autosave/collaboration.
- Marketplace payment, refund, and payout.
- Advanced anti-reverse-engineering.
- Enterprise policy.

## 2. Recommended five-week plan

### Week 1 - Contracts and risk prototypes

- Graph and node/edge contract.
- Stable node ID.
- Local project format.
- Basic canvas and Inspector.
- System Node Registry contract.
- TypeScript editor template and typings.
- Prototype JS sandbox on target Android device.
- Prototype scheduler branch/merge/loop.
- Prototype Android Camera/Bluetooth permission lifecycle.

Do not defer these four high-risk prototypes: canvas, sandbox, scheduler, Android
capability lifecycle.

### Week 2 - Development Runtime

- Graph validator.
- Scheduler state machine.
- Conditional edge.
- Activation and Merge.
- Controlled Loop.
- Custom TypeScript handler.
- HTTP Capability Broker.
- Pinned input and debug result.
- Formatter/linter/type-check.
- Cancellation and resource ceilings.

### Week 3 - Submission, Build, and Admin Review

- Registered Project and quota.
- Snapshot upload and immutable Revision.
- Automated validation.
- Workflow IR compiler.
- Candidate `.apkg` build.
- Sandbox test.
- Diff engine.
- Admin graph/source/diff viewer.
- Review decision.
- Artifact signing and Product Version.

### Week 4 - Marketplace and Android Launcher

- Product listing and Customer Library.
- Mock acquisition/Admin entitlement grant.
- Launcher login and Library sync.
- Download, verification, and atomic install.
- Runtime integration.
- Camera/Bluetooth adapter.
- Product-level permission UI.
- Customer credential flow.
- Customer Monitor and Stop.

### Week 5 - License and stabilization

- Entitlement validation.
- Signed Lease issue and refresh.
- Offline execution.
- Clock and expiry cases.
- Product update and rollback path.
- Basic analytics and audit lookup.
- End-to-end test on physical Android device.
- Resource leak, crash, cancellation, and tamper testing.
- UAT bug fixing.

If a sixth week exists, use it for payment, broader System Nodes, and UX polish.

## 3. Golden-path acceptance scenarios

| ID | Scenario | Expected result |
|---|---|---|
| UAT-01 | Build workflow with input, transform, conditional, merge, loop, and HTTP | Graph validates and development run succeeds |
| UAT-02 | Run a Custom Node with pinned input | Real handler runs in sandbox and result is inspectable |
| UAT-03 | Register, freeze, and submit | Revision is immutable and hash-verifiable |
| UAT-04 | Validate and build candidate | Candidate `.apkg`, reports, and hash are produced |
| UAT-05 | Reviewer inspects and approves | Decision references exact candidate hash |
| UAT-06 | Sign and publish | Signature verifies; unsigned candidate is not distributable |
| UAT-07 | Admin grants entitlement; Customer installs | Library and installed state are consistent |
| UAT-08 | Customer grants permission and binds credential | Broker allows only approved/granted scope |
| UAT-09 | Customer runs Product online | Scheduler, progress, and final state are correct |
| UAT-10 | Customer runs offline within Lease window | Execution starts without server dependency |
| UAT-11 | Customer revokes Camera while running | Later call fails with `PERMISSION_REVOKED` |
| UAT-12 | Customer stops Execution | No new downstream side effect starts |
| UAT-13 | Package is modified by one byte | Install/run is blocked |
| UAT-14 | Update requests a new capability | New consent is required; old grant is not inherited |
| UAT-15 | Lease expires while offline | Run is blocked and online refresh is requested |

## 4. Negative and security scenarios

- Forbidden import, `eval`, and dynamic import are rejected.
- Undeclared capability or domain is rejected.
- Zip slip, archive bomb, excessive file count, or compression ratio is rejected.
- Candidate hash changed after approval cannot be signed.
- Invalid signature or revoked signer blocks install.
- Runtime range mismatch blocks install/run.
- Product grant Allowed plus Android permission Denied still blocks native call.
- Credential disconnect creates a credential error without exposing token.
- Expired/revoked Entitlement cannot obtain a new Lease.
- Clock rollback requires online trust refresh.
- Loop and network limits stop abusive execution.
- Update verification failure preserves the active old version.

## 5. Definition of Done

- All UAT-must features have an acceptance test or an approved waiver.
- No open blocker/critical issue permits bypass of package verification, signing,
  credential isolation, permission, Entitlement, or License.
- Golden path succeeds on a physical Android device.
- Permission revoke, cancellation, and offline Lease are tested against actual
  Android lifecycle behavior.
- Audit can trace Revision -> Build -> Review -> Signature -> Product Version ->
  Execution.
- Known limitations are documented and do not contradict product claims.

## 6. Main risks

| Risk | Level | UAT mitigation |
|---|---|---|
| Canvas and Monaco integration | High | Prototype immediately with a large graph |
| TypeScript sandbox on Android | High | Select engine early; test timeout/cancel/memory |
| Scheduler branch/merge/loop correctness | High | Explicit state model and deterministic fixtures |
| Android permission lifecycle | High | Test device, foreground/background, mid-call revoke |
| OAuth redirect and Credential Vault | High | Implement one provider end-to-end first |
| Signing supply chain | High | Separate signer and bind approval to exact hash |
| Offline clock/revocation | Medium | Trusted time and bounded signed Lease |
| God Node and abuse | Medium | Capability/side-effect/complexity policy and review |
| Marketplace payment scope | Medium | Mock Entitlement so it cannot block the core path |

## 7. Decisions to carry into SDS

- Chosen Android JS engine and isolation model.
- Workflow IR schema and expression language.
- Concrete System Node list for UAT.
- Runtime persistence and crash recovery.
- Idempotency contract for side-effect nodes.
- Build/review queue and service-level targets.
- Signing key and rotation design.
- OAuth provider, Vault, and token refresh design.
- Device identity and emergency revocation policy.
- API boundaries and async event/message contracts.

## 8. Decisions to carry into Database Design

- Draft optimistic locking and Revision snapshots.
- Content-addressed source/asset/artifact storage.
- Submission/Build/Decision/version relationships.
- Entitlement and Lease consistency.
- Execution/Node Execution high-volume storage.
- Audit append model.
- Retention, partition, archive, and purge.
- Revocation lookup and caching.
- Credential metadata versus secret-store boundaries.

