# System Context

## 1. Product vision

A2MaTeX is a platform for creating, reviewing, distributing, and running
automation applications. It combines the accessibility of a visual workflow
builder with restricted custom code and a controlled mobile runtime.

The Publisher should be able to express real automation logic without building
an Android application, hosting a full execution environment, or distributing
credentials to Customers. The Customer should be able to install one trusted
Launcher, understand the permissions requested by each product, connect their
own accounts, and run purchased workflows online or for a bounded period offline.

## 2. Problems the platform solves

### For Creators and Publishers

- Building an automation product normally requires UI, backend, distribution,
  authentication, licensing, update, and security infrastructure.
- Visual tools are easy to start with but become restrictive when a workflow
  needs custom transformation or integration logic.
- Shipping unrestricted custom code to Customer devices creates security,
  credential, and policy risks.
- A Publisher needs a reviewable and versioned supply chain from source to the
  artifact actually distributed.

### For Customers

- Installing a separate APK for every small automation product is inconvenient
  and difficult to trust.
- Android grants permissions to an application, not to each workflow package.
- Customers need control over which product can use Camera, Bluetooth, files,
  network domains, and connected accounts.
- Customers need comprehensible execution progress without being forced to read
  source code or raw JSON logs.

### For the platform operator

- The platform must prevent a reviewed product from doing more at Runtime than
  was declared and approved.
- The platform needs enforceable plan limits without pretending it can control
  how many folders a user creates offline.
- The platform needs artifact integrity, version traceability, entitlement,
  bounded offline use, revocation, and auditable administration.

## 3. Core design principles

### 3.1 Local creativity, server-controlled publishing

A user may create any number of local projects. A free or paid plan limits
server resources instead:

- Registered Project Slots.
- Publish Requests.
- Cloud builds.
- Submission and artifact storage.
- Log retention.
- Cloud execution and future AI credits.

An unregistered local project may still be edited, formatted, linted, and tested.
It cannot receive a Product ID, build a production artifact, enter review, or be
published.

### 3.2 Review plus enforcement

Human review evaluates intent and context:

- What the code does.
- What data it sends.
- Which domains it contacts.
- Which credential and hardware capabilities it uses.
- Whether the behavior matches the product description.
- Whether side effects, loops, or request patterns are excessive.

Runtime enforcement checks every sensitive operation:

```text
Custom/System Node
  -> Platform SDK
  -> Capability Broker
  -> Package declaration
  -> Review approval
  -> Product-level grant
  -> Android OS permission
  -> License and runtime quota
  -> Native or network operation
```

Review does not grant unrestricted trust to a package.

### 3.3 Immutable release chain

```text
Mutable Draft
  -> Immutable Workflow Revision
  -> Submission
  -> Candidate .apkg + content hash
  -> Review Decision on exact hash
  -> Platform signature
  -> Published Product Version
```

Editing the Draft after submission does not alter the submitted Revision. If a
Reviewer requests changes, the Publisher creates and submits a new Revision.

### 3.4 Least privilege

A product may use only capabilities, domains, and credential slots that are:

1. declared by its package;
2. detected by validation;
3. approved in review;
4. granted by the Customer;
5. permitted by Android where applicable; and
6. allowed by current license and resource policy.

### 3.5 Bounded offline execution

Offline use is authorized by a signed License Lease with an `offlineUntil`
timestamp. The system does not promise both unlimited offline use and immediate
remote revocation; those goals conflict.

### 3.6 Observable but privacy-preserving

Execution telemetry should primarily contain:

- Product and version.
- Execution and node identifiers.
- Status and duration.
- Error code and retry count.
- Permission denial and resource-limit events.

The platform does not upload request bodies, response bodies, OCR content, file
content, tokens, email addresses, or precise location by default.

## 4. Actors

| Actor | Responsibilities | UAT status |
|---|---|---|
| Guest | Browse public product information | Included |
| Customer/Consumer | Acquire, install, authorize, configure, run, and monitor a product | Included |
| Creator/Developer | Build graph and custom code; run development tests | Included |
| Publisher | Register projects, submit versions, manage product listing and release | Included |
| Security Reviewer | Inspect graph, source, diff, capability, domain, and sandbox result | Included |
| Marketplace Moderator | Review listing/content policy | May be combined with Admin |
| Platform Admin | Operate projects, builds, products, entitlements, revocation, and policy | Included |
| Support Operator | Diagnose account/install/license issues without seeing secrets | Minimal |
| Finance Operator | Handle payments, refunds, payouts | After UAT |
| Team Member | Collaborate on Publisher projects with roles | After UAT |

## 5. End-to-end lifecycle

1. Creator makes a local project in Developer Studio.
2. Creator adds System Nodes and Custom TypeScript Nodes.
3. Creator runs a node with pinned input or runs the complete workflow.
4. Publisher registers the project on the platform.
5. Publisher freezes and uploads an immutable Revision.
6. Automated validation checks graph, source, imports, capabilities, domains,
   secrets, limits, and package safety.
7. Build Service compiles Workflow IR and creates a candidate `.apkg`.
8. Sandbox tests run with fake credentials and mocked capabilities.
9. Reviewer inspects the graph, source, diff, permissions, domains, risks, and
   sandbox result.
10. Signing Service signs the exact candidate hash approved by the Reviewer.
11. A Product Version is published in the Web Marketplace.
12. The Customer purchases the product or receives an Admin-granted entitlement.
13. Android Launcher downloads and verifies the `.apkg`.
14. Customer grants product permissions and connects required credentials.
15. Launcher obtains or refreshes a signed License Lease.
16. Runtime Agent verifies the package and lease, then executes the graph.
17. Customer Monitor shows progress and supports cancellation.
18. Platform and Launcher retain permitted logs, analytics, audit, and previous
    versions for update or rollback.

## 6. UAT success definition

UAT proves one complete, real golden path. It does not need marketplace payments,
multi-language runtimes, or a full security engine, but it must use:

- A real workflow graph.
- A real scheduler.
- Restricted TypeScript execution.
- Real `.apkg` packaging and verification.
- Real revision and review state.
- Real product-level permission checks.
- Real entitlement and signed lease checks.
- Real Android lifecycle behavior on at least one physical device.

## 7. Explicit exclusions from this documentation layer

- Table, column, index, migration, partition, and transaction design.
- Concrete service/class/module breakdown.
- Detailed REST/GraphQL/WebSocket contracts.
- Chosen JS engine and sandbox implementation.
- Infrastructure topology and cloud vendor selection.
- Final cryptographic key hierarchy and rotation procedure.

Those decisions belong to SDS, Database Design, API, and Threat Model documents.

