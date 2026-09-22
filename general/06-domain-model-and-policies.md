# Conceptual Domain Model and Platform Policies

This is a logical business model, not a database schema. Do not derive table,
column, index, or storage decisions directly from this file without the later
Database Design work.

## 1. Core concepts

| Concept | Meaning | Key relationships |
|---|---|---|
| Account | User identity and access status | Roles, sessions, Publisher/Customer profile |
| Publisher | Party that builds and releases Products | Projects, submissions, Products |
| Project | Registered development space | Draft, node sources, assets, Revisions |
| Workflow Draft | Current mutable state | Graph and source references |
| Workflow Revision | Immutable source snapshot | One-to-many Submissions/Build attempts |
| Node Definition | Versioned node contract | System, custom, private library |
| Node Source | Source text and hash tied to node/revision | Custom Node |
| Project Asset | Binary reference and content hash | Revision and package |
| Submission | Request to review one Revision | Build, reports, decision |
| Build | One candidate-generation attempt | Candidate Artifact and reports |
| Review Decision | Immutable decision on candidate hash | Reviewer and findings |
| Artifact | Candidate or signed `.apkg` | Hash, signature, storage reference |
| Product | Commercial/distribution identity | Listing and Product Versions |
| Product Version | Immutable published release | Signed Artifact and Runtime range |
| Entitlement | Customer's business right to Product | Plan, status, validity |
| License Lease | Signed device execution authorization | Product/version/device/offline window |
| Product Grant | Customer permission for one capability | Product/device/user/capability |
| Credential | Secret/token owned by Customer or Publisher | Provider and scopes |
| Credential Binding | Reference from product slot to Credential | Never contains raw token |
| Execution | One run of a Product Version | Lease, package hash, Node Executions |
| Node Execution | One node attempt in an Execution | Status, duration, retry, error |
| Audit Event | Administrative/security event | Actor, action, target, result |

## 2. Domain invariants

- A Draft is mutable; a Revision is immutable.
- A Submission references exactly one Revision.
- A Review Decision references exactly one candidate artifact hash.
- A signed Artifact must have the same content hash that was approved.
- A published Product Version references one immutable signed Artifact.
- An Execution records the Product Version, package hash, and Lease used at start.
- A graph stores credential binding metadata, never raw credential material.
- A product capability cannot exceed package declaration and review approval.
- Product-level permission cannot override a denied Android OS permission.
- A revoked or expired Entitlement cannot obtain a new valid Lease.
- Removing a Publisher plan must not delete an Artifact still required by an
  active Customer right or retention rule.

## 3. Main state models

### 3.1 Project

```text
LOCAL_ONLY -> REGISTERED -> ACTIVE
                         -> ARCHIVED
                         -> SUSPENDED
```

`LOCAL_ONLY` is a Studio state and may not exist on the server.

### 3.2 Submission

```text
SUBMITTED -> VALIDATING -> BUILDING -> SANDBOXING -> IN_REVIEW
    |            |           |            |             |
    +------------+-----------+------------+-----------> REJECTED
                                                     -> CHANGES_REQUESTED
                                                     -> ESCALATED
                                                     -> APPROVED -> SIGNING -> PUBLISHED
```

Infrastructure failure should remain distinguishable from product rejection.

### 3.3 Product Version

```text
DRAFT -> IN_REVIEW -> APPROVED -> PUBLISHED -> DEPRECATED
                                     |
                                     +-------> REVOKED
```

### 3.4 Entitlement

```text
PENDING -> ACTIVE -> EXPIRED
              |---> SUSPENDED -> ACTIVE
              |---> REVOKED
              |---> REFUNDED   (after payment scope)
```

### 3.5 Installed Product

```text
NOT_INSTALLED -> DOWNLOADING -> VERIFYING -> INSTALLED -> ACTIVE
                                  |             |          |
                                  +-> FAILED    +-> UPDATE_AVAILABLE
                                                        -> UPDATING
                                                        -> ACTIVE
```

### 3.6 Execution

```text
CREATED -> VALIDATING -> RUNNING -> SUCCEEDED
                         |-------> FAILED
                         |-------> CANCEL_REQUESTED -> CANCELLED
```

## 4. Business rules

| ID | Rule |
|---|---|
| BR-001 | Local project count is not plan-enforced; Registered Project Slots are |
| BR-002 | Node count is not a paywall; safety ceilings are technical policy |
| BR-003 | Submission always freezes an immutable Revision |
| BR-004 | Requested changes require a new Revision/Submission |
| BR-005 | Signing applies only to the exact approved candidate hash |
| BR-006 | Custom code uses only platform-approved imports |
| BR-007 | Network and native access use Capability Broker |
| BR-008 | Declared capabilities must cover detected capabilities |
| BR-009 | Arbitrary cycles are forbidden; loops use Loop Controller |
| BR-010 | Dynamic loops remain subject to hard limits |
| BR-011 | A node should have one primary observable side effect |
| BR-012 | Review approval never bypasses Runtime checks |
| BR-013 | Raw credentials are absent from graph, source snapshot, logs, and `.apkg` |
| BR-014 | Offline execution is valid only inside a signed Lease window |
| BR-015 | Capability increases in an update are not automatically granted |
| BR-016 | Published Artifact retention is independent of Publisher downgrade |
| BR-017 | Layout-only graph change is separated from security-relevant change |
| BR-018 | Production analytics exclude sensitive payload by default |

## 5. Configurable project and package limits

| Item | UAT default |
|---|---:|
| Graph JSON | 10 MB |
| Total source | 10 MB |
| One Custom Node source | 256 KB |
| One asset | 10 MB |
| Total assets | 50 MB |
| Compressed submission | 50 MB |
| Expanded submission | 100 MB |
| File count | 1,000 |
| Compression ratio | 20:1 maximum |

All values are policy configuration, not constants in business code.

## 6. Runtime hard-limit categories

- Total execution duration.
- Per-node timeout.
- Memory/accounted allocation.
- Network request count.
- Network request and response bytes.
- Response body size.
- Log bytes and event frequency.
- Loop iterations.
- Parallel node count.
- Retry count.
- Collected loop output bytes.
- Credential access frequency.
- Source, AST, and complexity at build time.

## 7. Retention baseline

| Data | UAT retention |
|---|---|
| Local autosave | Latest 20-50 snapshots |
| Development build | 7 days |
| Uploaded but unsubmitted snapshot | 7 days |
| Rejected Submission | 30-90 days |
| Submitted Revision | While Product remains active or policy requires |
| Published metadata and hash | Long-term |
| Active `.apkg` | Must be retained |
| Previous `.apkg` | Two latest versions |
| Development logs | 7 days |
| Production logs | 30 days |
| Audit/security events | At least 1 year |

Retention must account for active entitlement, investigation, legal hold,
revocation, and rollback needs.

## 8. Analytics boundary

### Development log

Can show sanitized per-run details to the Creator:

```text
OCR             Success    2.3s
Transform       Success     32ms
Google Sheets   Failed     1.1s
Error: HTTP 429
```

### Production aggregate

```text
Total runs
Success rate
P95 duration
Most failed node
Top error
Permission-denied rate
Resource-limit rate
```

### Production event fields

- Product and version.
- Execution ID.
- Node ID/type.
- Status.
- Start and end time.
- Duration.
- Error code.
- Retry count.
- Permission denial.
- Resource-limit violation.

### Not uploaded by default

- Request/response bodies.
- OCR or document content.
- File content.
- Token/secret.
- Email address.
- Precise location.

## 9. Error taxonomy

| Code/group | Meaning | Expected handling |
|---|---|---|
| `VALIDATION_*` | Graph, schema, type, or input invalid | Creator fixes project/input |
| `BUILD_*` | Compile, package, or sandbox failure | Creator fix or platform retry |
| `PACKAGE_INVALID` | Structure, hash, or signature invalid | Quarantine/redownload |
| `RUNTIME_INCOMPATIBLE` | Runtime range unsupported | Update Launcher or use another version |
| `ENTITLEMENT_INACTIVE` | No active business right | Renew or support |
| `LICENSE_REQUIRED` | Missing or expired Lease | Connect and refresh |
| `PERMISSION_DENIED` | Product or OS permission missing | Grant or open Android settings |
| `PERMISSION_REVOKED` | Permission removed during run | Error edge or stop |
| `CREDENTIAL_MISSING` | Credential slot unbound | Connect account |
| `CREDENTIAL_EXPIRED` | Token expired/revoked | Reconnect/refresh |
| `DOMAIN_NOT_ALLOWED` | Request outside allowlist | Block; new review if intentional |
| `RESOURCE_LIMIT` | Runtime quota exceeded | Stop; optimize workflow/policy |
| `EXECUTION_CANCELLED` | User/system cancellation | Terminal cancelled state |
| `EXTERNAL_SERVICE_ERROR` | Third-party error | Retry/error edge according to policy |
| `PLATFORM_UNAVAILABLE` | Platform dependency unavailable | Retry; use valid local Lease where possible |

Stable machine-readable codes must be separate from localized messages.

## 10. Database design handoff questions

The later Database Design should resolve:

- Aggregate and transaction boundaries.
- Which immutable objects are rows versus object-storage manifests.
- Optimistic-lock model for mutable Draft.
- Revision deduplication and content-addressed storage.
- Event/audit append strategy.
- Execution and Node Execution volume/partitioning.
- Retention and purge jobs.
- Entitlement and Lease consistency model.
- Version/revocation lookup and caching.
- Credential metadata versus secret-store boundary.

