# Product Module Feature Spec

**Status:** Draft for review; documentation only, not implemented  
**Module group:** 03 — Product, Submission, Review, and Product Release  
**Owner:** Backend; consumed by the Web Publisher Portal and later Submission, Release, and Catalog modules

## 1. Purpose and boundary

`Product` is the stable Marketplace identity for one Publisher-owned Studio Project. It is created
once and remains the parent identity across listing changes, submissions, reviews, and releases.

```text
PublisherAccount
└── Product ── Project
    ├── ProductListing       # separate feature
    ├── Submission[]         # separate feature
    └── ProductRelease[]     # separate feature
```

This module owns:

- the one-to-one Product-to-Project identity;
- Publisher ownership derived from the selected Project;
- Product lifecycle state and lifecycle authorization;
- Publisher/Admin Product list and detail reads;
- Publisher abandonment of an unpublished draft Product;
- Admin suspension and reactivation of a published Product;
- internal lifecycle operations used by the future Product Release module.

This module does not own:

- Product name, slug, description, category, tags, visibility, or compatibility metadata;
- Product media, Asset records, Cloudflare R2 objects, or presigned URLs;
- self-review data, Workflow Revision creation, or Submission state;
- automated scan findings or admin Submission review decisions;
- package building, package encryption, package/payload hashes, or release manifests;
- public Marketplace catalog queries;
- offers, orders, entitlement, download authorization, execution leases, or execution reports.

The Product module must not copy Project workflow content or Publisher profile data into Product.

## 2. MVP contract decisions for review

- One Project can own exactly one Product for its entire lifetime, including after Product archival
  or exceptional soft deletion. A Product is never recreated for the same Project.
- A Product belongs to the same `PublisherAccount` as its Project. Neither `projectId` nor
  `publisherAccountId` can change after creation.
- Product creation is an explicit Publisher Portal operation performed after the Publisher selects
  a cloud Project. It is not performed by Studio.
- Repeating creation for the same accessible Project is idempotent while its Product is not
  archived: return the existing Product ID without creating another record.
- Repeating creation for a Project whose Product is archived returns
  `409 Error.ProductArchived`; the archived Product is not restored or replaced.
- Product creation requires an active, non-deleted Project owned by the caller's active Publisher
  account. A Project Draft or Runtime IR is not required until Submission.
- Product has no editable business metadata. Product Listing owns all publisher-editable Marketplace
  presentation fields, so this module exposes no generic Product update endpoint.
- MVP statuses are `DRAFT`, `ACTIVE`, `SUSPENDED`, and `ARCHIVED`.
- A newly created Product is `DRAFT`.
- Only the Product Release module can transition `DRAFT` to `ACTIVE`, and only after an immutable
  Product Release has been published successfully.
- A Publisher may archive only its own `DRAFT` Product. Published or suspended Products cannot be
  archived because release and future customer-ownership history must remain addressable.
- Only Admin may suspend or reactivate a Product. Suspension is a Product-level distribution
  control; it does not delete the Product, Project, Submission, Release, or customer history.
- Blocking a Publisher does not rewrite Product statuses. Access policies and public Catalog queries
  must independently exclude a blocked Publisher where appropriate.
- `currentReleaseId` is not introduced by the Product migration. The Product Release feature adds
  that nullable column and its foreign key when `ProductRelease` exists.
- Asset-related relations and presigned URLs are post-MVP. The existing `Static`/Cloudflare R2
  storage path remains available to later package work but is not part of Product identity.

## 3. Data model

```text
Product
  id UUID PK
  publisherAccountId UUID FK PublisherAccount, required, immutable
  projectId UUID FK Project, required, immutable, unique
  status ProductStatus, required, default DRAFT
  statusReason TEXT, nullable
  suspendedAt TIMESTAMPTZ, nullable
  suspendedById UUID FK User, nullable
  createdAt TIMESTAMPTZ, required
  updatedAt TIMESTAMPTZ, nullable
  deletedAt TIMESTAMPTZ, nullable
  createdById UUID FK User, required
  updatedById UUID FK User, nullable
  deletedById UUID FK User, nullable
```

```text
ProductStatus = DRAFT | ACTIVE | SUSPENDED | ARCHIVED
```

All foreign keys use `NO ACTION`. Product has no public delete or restore operation; `deletedAt` and
`deletedById` are reserved for exceptional administrative repair or future retention policy.

Database integrity requirements:

- `projectId` is globally unique, not a partial unique index. Soft deletion does not make a Project
  eligible for another Product.
- Add a unique integrity constraint on `Project(id, publisherAccountId)` and a composite Product
  foreign key `(projectId, publisherAccountId)` to ensure that Product ownership cannot disagree
  with Project ownership. Keep the direct `publisherAccountId` foreign key to `PublisherAccount`.
- `SUSPENDED` requires non-null `statusReason`, `suspendedAt`, and `suspendedById`.
- Every non-`SUSPENDED` status requires those three suspension fields to be null.
- `statusReason` is normalized and validated by the application; it is not used for draft/archive
  notes.
- Add the following query index for owner lists:

```sql
CREATE INDEX "Product_publisherAccountId_status_idx"
ON "Product" ("publisherAccountId", status)
WHERE "deletedAt" IS NULL;
```

No listing, release, submission, media, or JSON snapshot column belongs in this table.

## 4. Lifecycle and invariants

Allowed transitions:

```text
[create] ──> DRAFT
DRAFT ─────> ACTIVE       # internal: first Product Release published
DRAFT ─────> ARCHIVED     # Publisher owner or unrestricted Admin
ACTIVE ────> SUSPENDED    # unrestricted Admin
SUSPENDED ─> ACTIVE       # unrestricted Admin after release eligibility check
```

All other transitions are rejected.

Lifecycle rules:

- `ARCHIVED` is terminal and readable.
- Archive is idempotent for an accessible `DRAFT` Product.
- An archive request for `ACTIVE` or `SUSPENDED` returns a stable conflict instead of hiding a
  Product that may have release or ownership history.
- Suspension requires a trimmed reason containing `1..2,000` characters.
- Suspension of a `DRAFT` or `ARCHIVED` Product is rejected.
- Suspending an already suspended Product is idempotent only when the caller is Admin; it preserves
  the original actor, timestamp, and reason.
- Reactivating an `ACTIVE` Product is idempotent.
- Reactivation must confirm through the Product Release module that the Product still has an
  eligible published current release. Until Product Release exists, no Product can legitimately be
  `ACTIVE` or `SUSPENDED`, so the `SUSPEND` and `REACTIVATE` lifecycle actions have no successful
  business case.
- Publishing a new release must not automatically reactivate a suspended Product.
- Archiving the related Project does not archive the Product. It prevents further Project mutation
  and future Submission eligibility through the Project/Submission rules, while existing Product
  history remains readable.

## 5. API contract

All routes use Bearer authentication, route permissions, strict Zod inputs, the global response
envelope, and the existing pagination format.

### 5.1 Common transport and schemas

Every successful JSON response is wrapped by the existing global interceptor:

```ts
type SuccessResponse<T> = {
  data: T;
  statusCode: number;
};
```

Domain and validation errors use the existing error response shape:

```ts
type ErrorResponse = {
  statusCode: number;
  message:
    | string
    | Array<{
        message: string;
        path: Array<string | number>;
      }>;
  details?: Record<string, string | number | boolean | null>;
};
```

All objects are strict: unknown request fields return `422`. UUIDs use the canonical UUID string
format accepted by the existing Zod contract. Timestamps are UTC ISO 8601 strings.

Reusable output shapes:

```ts
type ProductStatus = "DRAFT" | "ACTIVE" | "SUSPENDED" | "ARCHIVED";

type ProductProjectSummary = {
  id: string; // UUID
  name: string;
  status: "ACTIVE" | "ARCHIVED";
};

type ProductManagementOutput = {
  id: string; // UUID
  publisherAccountId: string; // UUID
  project: ProductProjectSummary;
  status: ProductStatus;
  statusReason: string | null;
  suspendedAt: string | null;
  createdAt: string;
  updatedAt: string | null;
};

type ProductIdParam = {
  productId: string; // UUID
};
```

`statusReason` and `suspendedAt` are non-null only when `status = 'SUSPENDED'`.

Common failures for every route:

| Condition                                              | Result                                       |
| ------------------------------------------------------ | -------------------------------------------- |
| Missing or invalid access token/session                | `401` existing authentication error          |
| No active role grants the exact method/path permission | `403 Forbidden`                              |
| Path, query, or body fails its strict schema           | `422` validation error with safe field paths |

### 5.2 Create or resolve Product identity

```http
POST /api/v1/products
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contract:

```text
method = POST
path   = /api/v1/products
scope  = null
role   = PUBLISHER
```

Strict request body:

```ts
type CreateProductInput = {
  projectId: string; // UUID
};
```

```json
{
  "projectId": "4a0906bf-7cf6-4c7d-83b7-976924fa399f"
}
```

Successful response: `201 Created` for both first creation and idempotent resolution of an existing
non-archived Product.

```json
{
  "data": {
    "id": "e5c89677-d139-47fe-9874-e3eceb574ec4"
  },
  "statusCode": 201
}
```

The caller supplies only `projectId`. The Backend derives `publisherAccountId`, `status`, audit
fields, and timestamps. It never accepts ownership or lifecycle fields from the request.

Route-specific failures:

| Condition                                                            | Result                          |
| -------------------------------------------------------------------- | ------------------------------- |
| Project is missing, archived, deleted, or owned by another Publisher | `404 Error.ProjectNotFound`     |
| Existing Product for this Project is archived                        | `409 Error.ProductArchived`     |
| Caller is not an eligible active Publisher                           | Existing Publisher policy error |
| Invalid UUID or unknown/missing body field                           | `422` validation error          |

Admin does not receive this permission. The endpoint never creates a Product on another
Publisher's behalf.

### 5.3 List Products for management

```http
GET /api/v1/products?page=1&limit=10&status=DRAFT
Authorization: Bearer <access-token>
```

Permission contracts:

```text
method = GET, path = /api/v1/products, scope = OWN  -> CUSTOMER, PUBLISHER
method = GET, path = /api/v1/products, scope = null -> ADMIN
```

Strict query:

```ts
type GetProductListInput = {
  page?: number; // positive integer, coerced from query string, default 1
  limit?: number; // positive integer, coerced from query string, default 10
  status?: ProductStatus;
};
```

Unknown query parameters return `422`. `OWN` access returns Products belonging to the caller's
non-deleted Publisher account. When an `OWN` caller has no Publisher account, the result is an empty
page rather than `404`. Unrestricted Admin access returns all matching non-deleted Products.

List order is `updatedAt DESC NULLS LAST, createdAt DESC, id DESC`.

Successful response: `200 OK`.

```json
{
  "data": {
    "items": [
      {
        "id": "e5c89677-d139-47fe-9874-e3eceb574ec4",
        "publisherAccountId": "e178ee97-27bd-4f23-b063-f0c7209e6a91",
        "project": {
          "id": "4a0906bf-7cf6-4c7d-83b7-976924fa399f",
          "name": "Daily Reward Collector",
          "status": "ACTIVE"
        },
        "status": "DRAFT",
        "statusReason": null,
        "suspendedAt": null,
        "createdAt": "2026-10-19T08:00:00.000Z",
        "updatedAt": null
      }
    ],
    "totalItems": 1,
    "page": 1,
    "limit": 10,
    "totalPages": 1
  },
  "statusCode": 200
}
```

```ts
type GetProductListOutput = {
  items: ProductManagementOutput[];
  totalItems: number; // non-negative integer
  page: number; // positive integer
  limit: number; // positive integer
  totalPages: number; // non-negative integer
};
```

### 5.4 Get Product detail for management

```http
GET /api/v1/products/{productId}
Authorization: Bearer <access-token>
```

Permission contracts:

```text
method = GET, path = /api/v1/products/:productId, scope = OWN  -> CUSTOMER, PUBLISHER
method = GET, path = /api/v1/products/:productId, scope = null -> ADMIN
```

Strict path parameters:

```ts
type ProductIdParam = {
  productId: string; // UUID
};
```

Successful response: `200 OK`.

```json
{
  "data": {
    "id": "e5c89677-d139-47fe-9874-e3eceb574ec4",
    "publisherAccountId": "e178ee97-27bd-4f23-b063-f0c7209e6a91",
    "project": {
      "id": "4a0906bf-7cf6-4c7d-83b7-976924fa399f",
      "name": "Daily Reward Collector",
      "status": "ACTIVE"
    },
    "status": "DRAFT",
    "statusReason": null,
    "suspendedAt": null,
    "createdAt": "2026-10-19T08:00:00.000Z",
    "updatedAt": null
  },
  "statusCode": 200
}
```

The management response does not expose audit user IDs, soft-delete fields, workflow content,
Publisher private data, Listing data, Submission data, or Release artifacts.

Missing, deleted, and cross-owner Products return `404 Error.ProductNotFound`. Invalid UUIDs return
`422` before the service runs.

### 5.5 Apply a Product lifecycle action

```http
PUT /api/v1/products/{productId}/lifecycle
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contracts:

```text
method = PUT, path = /api/v1/products/:productId/lifecycle, scope = OWN  -> PUBLISHER
method = PUT, path = /api/v1/products/:productId/lifecycle, scope = null -> ADMIN
```

Strict discriminated request body:

```ts
type UpdateProductLifecycleInput =
  | {
      action: 'ARCHIVE';
    }
  | {
      action: 'SUSPEND';
      reason: string; // trim, 1..2,000 characters
    }
  | {
      action: 'REACTIVATE';
    };
```

Each union member is strict. `reason` is required only for `SUSPEND`; it is rejected for `ARCHIVE`
and `REACTIVATE`. Unknown actions or fields return `422`.

Examples:

```json
{
  "action": "ARCHIVE"
}
```

```json
{
  "action": "SUSPEND",
  "reason": "Policy violation under review"
}
```

```json
{
  "action": "REACTIVATE"
}
```

Scope and action enforcement:

| Resolved permission access | Permitted actions | Additional rule |
| --- | --- | --- |
| `OWN` | `ARCHIVE` | Product must belong to the resolved active Publisher account. |
| Unrestricted `null` | `ARCHIVE`, `SUSPEND`, `REACTIVATE` | Product ownership is not used as a filter. |

The exact method/path permission is enforced by `AccessTokenGuard`. The controller passes the
resolved scopes to the Product service. The Product lifecycle policy then resolves a typed access
predicate and enforces the permitted action before any mutation:

```ts
type ProductLifecycleAccess =
  | {
      unrestricted: true;
    }
  | {
      unrestricted: false;
      publisherAccountId: string;
    };
```

An `OWN` caller attempting `SUSPEND` or `REACTIVATE` receives
`403 Error.ProductLifecycleActionDenied`, even though the caller has permission to reach the shared
lifecycle route. Product Release activation is not exposed through this endpoint.

Action behavior:

| Action | Allowed transition | Idempotent state | Behavior |
| --- | --- | --- | --- |
| `ARCHIVE` | `DRAFT -> ARCHIVED` | `ARCHIVED` | Preserve Product history and reject published/suspended Products. |
| `SUSPEND` | `ACTIVE -> SUSPENDED` | `SUSPENDED` | Record the trimmed reason, Admin actor, and timestamp. |
| `REACTIVATE` | `SUSPENDED -> ACTIVE` | `ACTIVE` | Require an eligible published current release and clear suspension fields. |

The repository performs each transition as an atomic conditional update using the expected source
status. An `OWN` update also includes `publisherAccountId` in the predicate. If no row changes, the
service performs one authorization-safe read to distinguish idempotent completion, not found, and
invalid lifecycle state.

Successful response for every action: `200 OK`.

```json
{
  "data": {
    "message": "Success"
  },
  "statusCode": 200
}
```

Route-specific failures:

| Condition | Result |
| --- | --- |
| Product is missing, deleted, or outside an `OWN` caller's ownership | `404 Error.ProductNotFound` |
| `OWN` caller requests `SUSPEND` or `REACTIVATE` | `403 Error.ProductLifecycleActionDenied` |
| `ARCHIVE` requested for `ACTIVE` or `SUSPENDED` | `409 Error.ProductCannotBeArchived` |
| `SUSPEND` requested for `DRAFT` or `ARCHIVED` | `409 Error.ProductCannotBeSuspended` |
| `REACTIVATE` requested for `DRAFT` or `ARCHIVED` | `409 Error.ProductCannotBeReactivated` |
| `REACTIVATE` requested without an eligible published current release | `409 Error.ProductReleaseUnavailable` |
| Invalid UUID, action, reason, or body shape | `422` validation error |
| `OWN` caller is no longer an eligible active Publisher | Existing Publisher policy error |

There is no Product rename/update, delete, restore, initial activation, or public Marketplace
endpoint in this feature.

### 5.6 Permission records and role assignment

Permission uniqueness follows the existing `(method, path, scope)` contract. The Product feature
requires the following permission records; `null` means unrestricted scope.

| Permission name | Module | Method | Stored path | Scope | Assigned roles |
| --- | --- | --- | --- | --- | --- |
| Create Product | Product | `POST` | `/api/v1/products` | `null` | `PUBLISHER`, `ADMIN` |
| List Own Products | Product | `GET` | `/api/v1/products` | `OWN` | `CUSTOMER`, `PUBLISHER` |
| List All Products | Product | `GET` | `/api/v1/products` | `null` | `ADMIN` |
| Read Own Product | Product | `GET` | `/api/v1/products/:productId` | `OWN` | `CUSTOMER`, `PUBLISHER` |
| Read Any Product | Product | `GET` | `/api/v1/products/:productId` | `null` | `ADMIN` |
| Manage Own Product Lifecycle | Product | `PUT` | `/api/v1/products/:productId/lifecycle` | `OWN` | `PUBLISHER` |
| Manage Any Product Lifecycle | Product | `PUT` | `/api/v1/products/:productId/lifecycle` | `null` | `ADMIN` |

No Product permission is assigned implicitly because a user has a Publisher account. Access still
requires an active role containing the matching permission record. When multiple active roles grant
the same route, unrestricted `null` takes precedence over `OWN` in the service access predicate.

## 6. Authorization and ownership behavior

- Create calls the existing `PublisherAccessPolicy.assertActivePublisher(userId)`. This requires an
  active User, active non-deleted Publisher account, active Publisher role, and acceptance of the
  current Publisher Terms.
- After Publisher eligibility is resolved, create queries an active, non-deleted Project using both
  `projectId` and the resolved `publisherAccountId`. Missing, archived, soft-deleted, and cross-owner
  Projects are externally indistinguishable.
- Create is an unscoped Publisher capability. Ownership is always derived server-side.
- `OWN` list/detail reads resolve the caller's non-deleted Publisher account regardless of
  `ACTIVE`/`BLOCKED` status, following the existing Project read pattern. A blocked former Publisher
  may inspect owned Product history through Customer `OWN` read permission.
- `OWN` lifecycle access requires an active Publisher and current Publisher Terms, filters by the
  resolved `publisherAccountId`, and permits only the `ARCHIVE` action.
- Unrestricted Admin lifecycle access does not resolve or validate the owner Publisher and permits
  `ARCHIVE`, `SUSPEND`, and `REACTIVATE`, while still obeying Product lifecycle rules.
- The route guard authorizes access to the shared lifecycle route; the Product lifecycle policy
  separately authorizes the requested action. Route access alone never authorizes every action.
- Admin list/detail access is unrestricted and does not resolve the Admin's Publisher account.
- Repositories consume a typed Product access predicate. They do not inspect role names, access-token
  payload structure, Redis permission-cache contents, or raw permission records.
- Missing IDs and cross-owner `OWN` access return the same not-found response to prevent resource
  enumeration.
- `UserStatus.BANNED` remains denied by the existing authentication/authorization layer.

The authoritative permission records and role assignments are defined in section 5.6. Permission
creation and role assignment use the existing permission APIs and must preserve every existing
role-permission ID.

## 7. Repository, transaction, and concurrency contract

- Product creation runs in one database transaction.
- Resolve Publisher eligibility before opening the Product transaction unless the policy itself must
  participate in a broader caller-owned transaction.
- Within the transaction, lock or conditionally select the owned Project before creating Product.
- Never trust a caller-supplied Publisher ID and never create a Product from an unrestricted Admin
  path.
- Concurrent create requests for the same Project are resolved by the database unique constraint.
  After a unique collision, perform an ownership-safe lookup and return the existing non-archived
  Product ID; return `Error.ProductArchived` for the archived match.
- Product lifecycle actions and internal release activation use atomic conditional updates on the
  expected source status.
- When a conditional mutation changes no row, make one authorization-safe read to distinguish
  idempotent completion, not found, and invalid lifecycle state without revealing cross-owner data.
- Do not use a Redis distributed lock. Database uniqueness and conditional writes are the durable
  concurrency boundaries.
- Do not query inside loops. Product list projections, including Project summaries, use a set-based
  relation query.
- Do not perform object-storage, queue, Runtime, or other network calls inside Product transactions.

## 8. Internal integration contracts

The future Submission module consumes Product through a narrow trusted read that verifies:

- Product exists and is not soft-deleted;
- Product belongs to the active submitting Publisher;
- Product is not `ARCHIVED` or `SUSPENDED`;
- Product references the submitted Project;
- the selected Project remains active and submission-eligible.

The future Product Release module owns release persistence and calls Product lifecycle operations in
the same transaction:

```ts
markActiveAfterPublishedRelease({
  productId,
  releaseId,
  actorId,
}): Promise<void>
```

That operation must reject `ARCHIVED` or `SUSPENDED` Products. The Product Release feature will add
`currentReleaseId`, verify that the release belongs to the same Product and is `PUBLISHED`, then set
both `Product.status = ACTIVE` and `Product.currentReleaseId = releaseId` atomically.

Catalog, Offer, Entitlement, and Download Authorization modules must not infer availability from
Product status alone. Their queries must also evaluate the current release and their own domain
rules.

## 9. Stable failure semantics

| Condition                                                             | Result                                      |
| --------------------------------------------------------------------- | ------------------------------------------- |
| Missing, soft-deleted, or cross-owner Product                         | `404 Error.ProductNotFound`                 |
| Missing, archived, soft-deleted, or cross-owner Project during create | `404 Error.ProjectNotFound`                 |
| Existing archived Product for the selected Project                    | `409 Error.ProductArchived`                 |
| `OWN` lifecycle caller requests `SUSPEND` or `REACTIVATE`              | `403 Error.ProductLifecycleActionDenied`    |
| Archive requested for `ACTIVE` or `SUSPENDED` Product                 | `409 Error.ProductCannotBeArchived`         |
| Suspend requested for non-`ACTIVE` Product                            | `409 Error.ProductCannotBeSuspended`        |
| Reactivate requested for a status other than `SUSPENDED`/`ACTIVE`     | `409 Error.ProductCannotBeReactivated`      |
| Reactivation without an eligible current release                      | `409 Error.ProductReleaseUnavailable`       |
| Invalid UUID, status filter, body shape, or reason                    | `422` from the global Zod validation pipe   |
| Inactive/blocked/unqualified Publisher                                | Existing stable Publisher policy error      |
| Missing route permission or insufficient scope                        | `403` from the existing authorization layer |

Safe error responses may include a current lifecycle status only after ownership/unrestricted access
has been established. They must not include Project content, Publisher private data, Submission
snapshots, package identifiers, or release secrets.

## 10. Privacy, logging, and operational behavior

- Do not add routine success logs for Product create, list, detail, or lifecycle actions.
- Centralized request/error logging covers expected domain failures.
- Admin suspension/reactivation may log Product ID, actor ID, old/new status, and a stable action
  code. Do not log the free-text reason; store it only in Product and the future Audit Log.
- Never log Project workflow content, Runtime IR, Publisher Terms payloads, access tokens, private
  profile fields, or future release/package content.
- Product mutations set the appropriate audit actor and timestamp fields.
- Product status changes that later publish durable jobs must use the operations/outbox contract;
  this initial identity module emits no external events.

## 11. Acceptance criteria

The feature is conformant when the following behaviors are demonstrated:

| Area             | Required behavior                                                                                                     |
| ---------------- | --------------------------------------------------------------------------------------------------------------------- |
| Identity         | Exactly one immutable Product identity exists per Project under concurrent creation.                                  |
| Ownership        | Product ownership always equals Project ownership and cannot be supplied or changed by clients.                       |
| Idempotency      | Repeated create returns the existing non-archived Product; repeating an action in its idempotent state succeeds.       |
| Publisher policy | Only a currently qualified Publisher can create or archive an owned draft Product.                                    |
| Read access      | Owner `OWN` and Admin unrestricted reads are isolated; cross-owner access is indistinguishable from not found.        |
| Lifecycle        | Only the approved transitions occur; `ARCHIVED` is terminal and suspension fields remain consistent.                  |
| Release boundary | No Product becomes `ACTIVE` without a successfully published Product Release transaction.                             |
| Project boundary | Archived Project does not erase Product history, but cannot create a new Product or support a future Submission.      |
| Data exposure    | Management responses contain identity/lifecycle data and Project summary only.                                        |
| Performance      | Lists and relation projections are set-based, paginated, deterministically ordered, and contain no per-row queries.   |
| Security         | No client can select another Publisher, enumerate cross-owner records, or mutate status through a generic update API. |
| Operations       | Expected mutations use database constraints/conditional writes and add no Redis lock or storage/network call.         |

## 12. Deferred features

- Product Listing and its editable presentation metadata.
- Category, Tag, and featured placement.
- Product media and all Asset-related relations.
- Presigned upload/download URLs.
- Submission, automated scan, review, and Workflow Revision orchestration.
- Product Release persistence, `currentReleaseId`, package creation, encryption, and integrity data.
- Public catalog/search APIs.
- Product Offer, acquisition, entitlement, order, payment, download authorization, and execution
  authorization.
- Product transfer between Publishers or Projects.
- Publisher team ownership.
- Product hard-delete, restoration, or reuse of a Project after archival.
