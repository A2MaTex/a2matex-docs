# The Four Platforms

## 1. Product boundary summary

| Product | Primary users | Owns | Does not own |
|---|---|---|---|
| Developer Studio | Creator, Publisher | Authoring, local persistence, validation feedback, development execution, submission initiation | Production signing, Customer permissions, entitlement |
| Web Marketplace | Guest, Customer, Publisher, Reviewer, Admin | Catalog, Customer library, Publisher operations, protected review/admin surfaces | On-device execution |
| Android Launcher | Customer | Login, library sync, download/install, consent, credential connection, run/stop UI | Workflow compilation and production signing |
| Runtime Agent | Launcher/platform | Package verification, scheduling, sandboxing, capability enforcement, execution events | Marketplace UI and authoring UX |

Shared backend capabilities are described at the end of this document. They are
not counted as a fifth product.

## 2. Developer Studio

### 2.1 Purpose

Developer Studio is an offline-first IDE for building automation products. It
must support serious workflow development rather than acting as a mock prototype.

### 2.2 Main surfaces

- Project explorer.
- Visual graph canvas.
- Node palette.
- Node inspector.
- Monaco-based TypeScript editor.
- Input/output/config schema editor.
- Test data and pinned-input editor.
- Run result, validation, and diagnostic panels.
- Project registration and submission screens.

### 2.3 Project layout

The conceptual local format is:

```text
project/
├── project.json
├── workflow.json
├── nodes/
│   ├── <node-id>.ts
│   └── <node-id>.ts
├── assets/
├── tests/
└── .a2matex/
    ├── local-state.db
    └── revision-journal/
```

Exact filenames may change in SDS. The important boundary is that graph metadata,
source, assets, and local editor state are separable.

### 2.4 Required UAT capabilities

- Create, open, rename, and save a local project.
- Add, delete, copy, and move nodes.
- Connect typed ports with edges.
- Add System Nodes from a platform registry.
- Create Custom TypeScript and Custom Integration Nodes.
- Define input, config, and output schema.
- Configure conditional edges and activation policies.
- Configure controlled loops.
- Format, lint, and type-check custom code.
- Run a complete workflow.
- Run one node with pinned input and mocked dependencies.
- Inspect input, output, error, duration, and execution state per node.
- Register a project and display plan quota.
- Freeze and submit an immutable revision.

### 2.5 Offline and server behavior

- Editing and local autosave do not require network.
- The Studio must not send a network request after every keystroke.
- The server stores at most one mutable synchronized Draft per Registered Project
  during UAT.
- Server Revisions are created only for explicit revision, submission, build, or
  publication events.
- Later cloud collaboration may use JSON Patch and periodic snapshots, but it is
  not required for UAT.

### 2.6 TypeScript boundary

The platform owns:

- `tsconfig`.
- TypeScript and SDK version.
- Runtime target.
- Build command and bundler.
- Approved utility libraries.
- Runtime global objects.

The Publisher writes only node logic and schema. Arbitrary npm packages,
filesystem access, process execution, dynamic import, and direct native calls are
not allowed in UAT.

### 2.7 Private Node Library

A minimal local-only Private Node Library is optional for UAT:

1. Save a custom node definition from Project A.
2. Assign a semantic version.
3. Add that version to Project B.
4. Pin the exact version.
5. Upgrade only when the Creator explicitly chooses to do so.

It is a reusable node definition, not a nested workflow. Cloud sync, sharing, and
a public Node Marketplace are later scope.

## 3. Web Marketplace

### 3.1 Purpose

The Web Marketplace is the web-facing product for discovery, acquisition,
Publisher operations, and protected platform operations.

### 3.2 Public and Customer surfaces

- Product catalog.
- Search and filter.
- Product details and version information.
- Publisher identity.
- Declared permissions, domains, and credential requirements.
- Acquisition or purchase action.
- Customer web library.
- Entitlement status and expiry.
- Support and product status notices.

For UAT, purchase may be replaced by Admin-granted entitlement.

### 3.3 Publisher surfaces

- Publisher profile and plan.
- Registered Project list and slot usage.
- Submission history.
- Build and validation result.
- Review status and findings.
- Product and version management.
- Listing metadata.
- Basic execution aggregates.

### 3.4 Reviewer/Admin surfaces

- Visual graph viewer.
- Source viewer and source diff.
- Semantic graph diff and visual-only diff.
- Capability, domain, credential, and side-effect summary.
- Automated validation and sandbox reports.
- Approve, reject, request changes, or escalate.
- Product/version/account/device/certificate revocation.
- Entitlement grant and revoke for UAT.
- Audit and operational lookup.

### 3.5 Admin is not a separate product

Admin Portal is a role-protected section of the Web Platform. It may be deployed
separately later, but the product model still has four products.

## 4. Android Launcher

### 4.1 Purpose

Android Launcher is the trusted host application installed by the Customer. All
products run within it through the shared Runtime Agent.

### 4.2 Required UAT capabilities

- Authenticate and maintain Customer session.
- Synchronize Customer Library.
- Show entitlement and installed/update state.
- Download and resume `.apkg` retrieval.
- Verify size, hash, signature, product identity, Runtime compatibility, and
  revocation state.
- Store packages in private application storage.
- Show permission purpose before use.
- Manage product-level grants.
- Request Android OS permission at the relevant action.
- Connect, disconnect, and rebind Customer credentials.
- Start and stop an execution.
- Show human-friendly progress and activity.
- Refresh a License Lease and run offline within its window.
- Update a product without breaking the current installed version.
- Retain recent versions for rollback according to policy.

### 4.3 What Launcher must not do

- Install a Publisher-provided APK.
- Execute `.apkg` with `exec()`.
- Expose raw OAuth tokens to the workflow or UI.
- Grant a capability that the package did not declare and review did not approve.
- Assume Android permission means every installed product may use it.

### 4.4 Customer Monitor

Default progress is expressed in business language:

```text
Invoice Scanner is running

✓ Image received
✓ Invoice content read
● Sending 12 rows to Google Sheets
○ Complete

[Stop]
```

Activity detail may show:

- Camera opened.
- Image processed.
- Connected to `sheets.googleapis.com`.
- Updated 12 rows.

It must not show raw tokens or full sensitive payloads.

## 5. Runtime Agent

### 5.1 Purpose

Runtime Agent is the shared execution engine. It is conceptually separate even
when packaged as a module inside Android Launcher.

### 5.2 Responsibilities

- Parse package structure and manifest.
- Verify hashes, signature, Runtime range, and revocation.
- Verify signed License Lease.
- Load Workflow IR.
- Resolve versioned System Node handlers.
- Load custom script modules into a restricted JS sandbox.
- Evaluate input mapping and conditional expressions.
- Schedule ready nodes.
- Enforce activation, merge, loop, retry, timeout, and cancellation policies.
- Route network and hardware through Capability Broker.
- Resolve credential references without exposing raw secrets unnecessarily.
- Enforce resource quotas.
- Emit progress and execution events.
- Write sanitized local logs and permitted aggregate telemetry.

### 5.3 Runtime is not trusted by package content

The Runtime must recheck sensitive operations even if the package passed review.
Package declarations are inputs to enforcement, not a reason to bypass it.

## 6. Shared platform capabilities

| Capability domain | Responsibility |
|---|---|
| Identity and Access | Account, role, session, block status, device identity |
| Publisher and Project | Publisher profile, plan, Registered Project Slots |
| Revision and Submission | Immutable snapshot, content hash, submission state |
| Build and Validation | Graph/source checks, Workflow IR, candidate package, sandbox |
| Review and Policy | Findings, risk summary, decision, policy version |
| Signing and Artifact | Exact-hash signing, artifact metadata, revoke, retention |
| Product and Catalog | Product, version, listing, update channel |
| Commerce and Entitlement | Acquisition, plan, validity, suspension, revoke |
| License | Signed lease issue, refresh, device/version/offline binding |
| Audit and Analytics | Security/admin audit and privacy-preserving aggregates |
| Object Storage | Snapshot, asset, candidate, signed artifact, retained versions |

