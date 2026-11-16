# Product Module Feature Spec

**Status:** Approved for implementation; existing Backend lifecycle requires revision
**Module group:** 03 — Product, Submission, Review, and Product Release  
**Owner:** Backend; consumed by the Web Publisher Portal, Submission, Release, and Catalog modules

## 1. Purpose and boundary

`Product` is the Publisher-owned editable Marketplace aggregate for one Studio Project. It combines
the stable Product identity with the Product's draft presentation fields. A Publisher creates and
saves this aggregate before submission. Creating or editing it does not make it official or public.

```text
PublisherAccount
└── Product ── Project
    ├── ProductMedia[] ── Static
    ├── Submission[]         # separate feature
    └── ProductRelease[]     # separate feature
```

This module owns:

- the one-to-one Product-to-Project identity;
- Publisher ownership derived from the selected Project;
- Publisher-authored name, slug, descriptions, usage instructions, and ordered media references;
- draft creation and partial draft saving;
- Product lifecycle state and lifecycle authorization;
- Publisher/Admin Product list and detail reads;
- Publisher activation, reversible archival, and reactivation of an owned Product;
- Admin suspension and reactivation of a published Product;
- internal lifecycle operations used by the Product Release module.

This module does not own:

- Static upload/processing or Cloudflare R2 objects; it stores only validated `staticId` references;
- public Marketplace Product list/detail projections, which require Submission and Product Release;
- Category, Tag, ProductTag, required-app classification, or connector taxonomy;
- Submission review context, Workflow Revision creation, or Submission state;
- automated scan findings or admin Submission review decisions;
- package building, package encryption, package/payload hashes, or release manifests;
- advanced Catalog ranking, recommendations, and full-text infrastructure;
- offers, orders, entitlement, download authorization, execution leases, or execution reports.

The Product module must not copy Project workflow content or Publisher profile data into Product.

Related authoritative contracts:

- [Product Release](./product_release.md) owns Release build/publication, history, and pointer-only
  current-Release selection.
- [Product Offer](../commerce_entitlement_execution/product_offer.md) owns commercial configuration,
  Offer readiness, and the sliding-window state used by Product activation.
- This Product specification owns Publisher lifecycle, Admin moderation, Marketplace Product API
  projections, and the transaction that coordinates activation/reactivation readiness.

## 2. MVP contract decisions for review

- One Project can own exactly one Product for its entire lifetime, including after Product archival
  or exceptional soft deletion. A Product is never recreated for the same Project.
- A Product belongs to the same `PublisherAccount` as its Project. Neither `projectId` nor
  `publisherAccountId` can change after creation.
- The Web Publisher Portal creates Product as `DRAFT` before submission. Studio does not create it.
- `POST /products` creates Product and its optional initial media references atomically. Exactly one
  request may create the Product for a Project; a repeated or concurrent loser receives a stable
  conflict and is never retried as transient work.
- `PATCH /products/:productId` saves the Publisher's editable working copy with MVP last-write-wins
  semantics. It never changes lifecycle status or the currently published release.
- Draft saves may be incomplete. Submission applies the complete publishing validation contract.
- Product creation requires an active, non-deleted Project owned by the active Publisher.
- An existing `ARCHIVED` or `SUSPENDED` Product cannot receive a new Submission. An `ACTIVE` Product
  can receive an update Submission while its currently published release remains official.
- Product Listing is not a separate MVP entity or route. "Listing information" means the editable
  presentation fields stored on Product.
- Product media is an ordered set of image references. Upload happens first through the existing
  Static/R2 image flow; Product requests contain metadata and `staticId`, never image bytes or R2 keys.
- Static is shared storage metadata and does not carry Product or Publisher ownership. Product
  create/update may attach any completed, non-deleted image Static that passed the existing image
  pipeline. The API still accepts only `staticId`; it never accepts object keys or remote URLs.
- Category, Tag, ProductTag, required apps, and connector taxonomy are deferred. Current basic
  Android actions provide no stable taxonomy worth maintaining. Release capabilities remain
  technical release data and are not Publisher-entered Product classification.
- MVP statuses are `DRAFT`, `ACTIVE`, `SUSPENDED`, and `ARCHIVED`.
- Do not persist `READY_TO_PUBLISH` or another readiness status. Activation readiness is derived by
  the Publisher lifecycle transaction from Product, current Release, Offer, and Publisher state.
- A newly created Product is `DRAFT`. Submission, scan completion, admin approval, or a
  queued/building release does not by itself change that status.
- Admin approval authorizes the build/publish pipeline; it does not create the Product and does not
  make it public by itself.
- Release publication leaves Product `DRAFT` and `currentReleaseId` unchanged. The Publisher may
  select a `PUBLISHED` Release and configure Offers in either order while the Product remains
  private. A separate Publisher `ACTIVATE` lifecycle transaction validates the selected Release and
  Offers, then transitions `DRAFT` to `ACTIVE` atomically.
- If the first Submission is rejected or changes are requested, the editable Product remains
  `DRAFT` and absent from the public catalog. A later corrected Submission snapshots it again.
- If an update Submission for an `ACTIVE` Product is rejected, requests changes, or fails build, the
  currently published release and approved Product snapshot remain unchanged.
- Every Publisher-authored change that would alter the official store representation or executable
  release requires a new Submission and admin review in the MVP. This includes workflow/Runtime IR,
  release version/compatibility, Product text, slug, usage instructions, and Product media.
- Submission freezes the Product fields, ordered media, Workflow Revision, and Release request as
  immutable snapshots. Marketplace reads the snapshot behind `currentReleaseId`; editing Product
  never overwrites the live store representation.
- A Publisher may archive only its own `ACTIVE` Product. `ARCHIVED` is a reversible Publisher
  unlisting state: it hides the Product and blocks new purchases without deleting Release, Offer,
  Order, or Entitlement history. Existing entitled customers retain access while the current Release
  remains eligible.
- The owner Publisher may reactivate only its own `ARCHIVED` Product. Admin may suspend only an
  `ACTIVE` Product and may reactivate only a `SUSPENDED` Product. Publisher operations can never
  override Admin suspension, and Admin moderation does not override Publisher archival.
- Blocking a Publisher does not rewrite Product statuses. Access policies and public Catalog queries
  must independently exclude a blocked Publisher where appropriate.
- Product owns nullable `currentReleaseId`; the Product Release migration adds its foreign key.
  Clients cannot write it. The Product Release module's controlled selection operation is the only
  owner of current-release changes.
- General Asset relations and presigned URLs remain post-MVP. Product images use the existing
  Static/Cloudflare R2 path because expected files are small and already supported by Backend upload.
- Product Offer adds internal `offerChangeTimestamps` to Product for its bounded sliding window.
  Product create/update, Submission, Review, and Release selection never write or expose it. The
  Publisher `ACTIVATE` lifecycle transaction records the initial timestamp through the injected
  `ProductOfferService`; Publisher/Admin reactivation and Release switching do not change it. Offer
  price/duration changes remain outside Submission and never rebuild a Product Release.

## 3. Data model

```text
Product
  id UUID PK
  publisherAccountId UUID FK PublisherAccount, required, immutable
  projectId UUID FK Project, required, immutable, unique
  status ProductStatus, required, default DRAFT
  currentReleaseId UUID, nullable, internal-only; Product Release migration owns the FK
  offerChangeTimestamps TIMESTAMPTZ[], required, default empty, internal-only
  name VARCHAR(255), nullable while draft
  slug VARCHAR(255), nullable while draft
  shortDescription VARCHAR(500), nullable while draft
  description TEXT, nullable while draft
  usageInstructions TEXT, nullable
  statusReason TEXT, nullable
  suspendedAt TIMESTAMPTZ, nullable
  suspendedById UUID FK User, nullable
  createdAt TIMESTAMPTZ, required
  updatedAt TIMESTAMPTZ, nullable
  deletedAt TIMESTAMPTZ, nullable
  createdById UUID FK User, required
  updatedById UUID FK User, nullable
  deletedById UUID FK User, nullable

ProductMedia
  id UUID PK
  productId UUID FK Product, required
  staticId UUID FK Static, required
  type ProductMediaType, required
  sortOrder INTEGER, required
  altText VARCHAR(500), nullable
  createdAt TIMESTAMPTZ, required
  updatedAt TIMESTAMPTZ, nullable
  deletedAt TIMESTAMPTZ, nullable
```

```text
ProductStatus = DRAFT | ACTIVE | SUSPENDED | ARCHIVED
ProductMediaType = ICON | COVER | GALLERY
```

All foreign keys use `NO ACTION`. Product has no public delete operation; `deletedAt` and
`deletedById` are reserved for exceptional administrative repair and the platform retention policy.

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
- `slug` is case-insensitively unique when non-null and non-deleted. Submission requires it.
- Active Product media rows are unique by `(productId, staticId)`.
- A Product can have at most one active `ICON` and one active `COVER`; `GALLERY` may contain many
  ordered images.
- Product media accepts only completed, non-deleted Static image records that passed the image
  pipeline. Static is shared and has no Publisher ownership. Product never accepts raw object keys
  or remote URLs.
- Removing ProductMedia from the working draft removes only the association. Static cleanup must not
  delete an image referenced by any immutable Submission or published Product Release.
- Add the following query index for owner lists:

```sql
CREATE INDEX "Product_publisherAccountId_status_idx"
ON "Product" ("publisherAccountId", status)
WHERE "deletedAt" IS NULL;
```

No release artifact, Workflow Revision content, or Submission snapshot belongs in Product.

## 4. Lifecycle and invariants

Allowed transitions:

```text
[create] ──> DRAFT
DRAFT ─────> ACTIVE       # owner Publisher ACTIVATE after readiness validation
ACTIVE ────> ARCHIVED     # owner Publisher ARCHIVE
ARCHIVED ──> ACTIVE       # owner Publisher REACTIVATE after readiness validation
ACTIVE ────> SUSPENDED    # Admin SUSPEND
SUSPENDED ─> ACTIVE       # Admin REACTIVATE after readiness validation
```

All other transitions are rejected.

Lifecycle rules:

- `[create] -> DRAFT` occurs through the Publisher Product creation API.
- A `DRAFT` Product remains a draft until activation; Publisher archival is not draft deletion.
- Archive is idempotent for an owned `ACTIVE` or already `ARCHIVED` Product and retains
  `currentReleaseId`, Offers, Releases, Orders, and Entitlements.
- `ARCHIVED` is absent from Marketplace and unavailable for new purchases, but existing unexpired
  Entitlements may continue download and execution against its eligible current Release.
- Suspension requires a trimmed reason containing `1..2,000` characters.
- Suspension of a `DRAFT` or `ARCHIVED` Product is rejected.
- Suspending an already suspended Product is idempotent only when the caller is Admin; it preserves
  the original actor, timestamp, and reason.
- Publisher activation and both reactivation paths must confirm through Product Release and the
  injected `ProductOfferService.assertActivationReady(productId)` that the Product has an eligible
  published current Release and at least one eligible subscription Offer.
- Initial `DRAFT -> ACTIVE` records the first Offer sliding-window timestamp. Publisher
  `ARCHIVED -> ACTIVE` and Admin `SUSPENDED -> ACTIVE` do not consume a slot because they do not
  change the commercial collection.
- Publishing a new release must not automatically reactivate a suspended Product.
- Archiving the related Project does not archive the Product. It prevents further Project mutation
  and subsequent Submission eligibility through the Project/Submission rules, while existing Product
  history remains readable.

## 5. API contract

Management and mutation routes use Bearer authentication and route permissions. Public Marketplace
routes use `@IsPublic()` with no Permission record. All routes use strict Zod inputs, the global
response envelope, and the existing pagination format.

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

type ProductMediaType = "ICON" | "COVER" | "GALLERY";

type ProductMediaInput = {
  staticId: string; // UUID
  type: ProductMediaType;
  sortOrder: number; // non-negative integer
  altText?: string | null; // trimmed, maximum 500 characters
};

type ProductMediaOutput = ProductMediaInput & {
  id: string; // UUID
};

type ProductOfferOutput = {
  id: string; // UUID
  type: "ONE_TIME" | "SUBSCRIPTION";
  priceAmount: string; // system-balance BIGINT serialized as a decimal string
  durationDays: number | null;
};

type ProductManagementOutput = {
  id: string; // UUID
  publisherAccountId: string; // UUID
  project: ProductProjectSummary;
  status: ProductStatus;
  name: string | null;
  slug: string | null;
  shortDescription: string | null;
  description: string | null;
  usageInstructions: string | null;
  media: ProductMediaOutput[];
  currentReleaseId: string | null;
  statusReason: string | null;
  suspendedAt: string | null;
  createdAt: string;
  updatedAt: string | null;
};

type ProductManagementDetailOutput = ProductManagementOutput & {
  offers: ProductOfferOutput[];
};

type ProductIdParam = {
  productId: string; // UUID
};
```

`statusReason` and `suspendedAt` are non-null only when `status = 'SUSPENDED'`. Media order is
deterministic: `type`, then `sortOrder`, then `id`.

Common failures for authenticated management and mutation routes:

| Condition                                              | Result                                       |
| ------------------------------------------------------ | -------------------------------------------- |
| Missing or invalid access token/session                | `401` existing authentication error          |
| No active role grants the exact method/path permission | `403 Forbidden`                              |
| Path, query, or body fails its strict schema           | `422` validation error with safe field paths |

### 5.2 Create a Product draft

```http
POST /api/v1/products
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contract:

```text
method = POST, path = /api/v1/products, scope = null -> PUBLISHER
```

Strict request body:

```ts
type CreateProductInput = {
  projectId: string; // UUID
  name?: string | null; // trim, 1..255 when present
  slug?: string | null; // normalized lowercase slug, 1..255 when present
  shortDescription?: string | null; // trim, 1..500 when present
  description?: string | null; // trimmed non-empty string when present
  usageInstructions?: string | null;
  media?: ProductMediaInput[]; // complete initial ordered set, default []
};
```

Only `projectId` is required for a draft save. The Backend derives Publisher ownership, status, and
audit fields. It validates all media references in one set-based query and creates
Product plus ProductMedia rows atomically.

First creation returns `201 Created` with only the new Product ID:

```ts
type CreateProductOutput = {
  id: string; // UUID
};
```

If another request already created a Product for the Project, the loser returns
`409 Error.ProductAlreadyExists`. This is a terminal business conflict, including under queued
delivery; it must not be classified as a transient failure or retried indefinitely. Clients
use `GET /products/{productId}` to fetch the complete Product and its relations.

Lifecycle and moderation failures:

| Condition                                                                            | Result                           |
| ------------------------------------------------------------------------------------ | -------------------------------- |
| Project is missing, archived, deleted, or cross-owner                                | `404 Error.ProjectNotFound`      |
| Existing non-archived Product for the Project                                        | `409 Error.ProductAlreadyExists` |
| Existing Product is archived                                                         | `409 Error.ProductArchived`      |
| Slug is already reserved by another Product                                          | `409 Error.ProductSlugConflict`  |
| Media contains an unavailable, unprocessed, non-image, or duplicate Static reference | `422 Error.ProductMediaInvalid`  |
| Caller is not an eligible active Publisher                                           | Existing Publisher policy error  |

### 5.3 Update a Product draft

```http
PATCH /api/v1/products/{productId}
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contract:

```text
method = PATCH, path = /api/v1/products/:productId, scope = OWN -> PUBLISHER
```

Strict request body:

```ts
type UpdateProductInput = {
  name?: string | null;
  slug?: string | null;
  shortDescription?: string | null;
  description?: string | null;
  usageInstructions?: string | null;
  media?: ProductMediaInput[]; // when present, replaces the complete draft media set
};
```

At least one mutable field is required. Omitted fields remain unchanged; explicit `null` clears an
optional draft field; `media: []` clears all draft media. The
request may update `DRAFT` or `ACTIVE` Products. It cannot update ownership, Project, status,
`currentReleaseId`, audit fields, Submission snapshots, or release data.

The update condition includes Product ID, owner, and allowed lifecycle state. Product fields and the
complete media replacement commit in one transaction using MVP last-write-wins behavior. Success
returns `200 OK` with the standard success message. `GET /products/{productId}` owns fetching the
complete record and preloading its related entities.

Route-specific failures:

| Condition                                   | Result                          |
| ------------------------------------------- | ------------------------------- |
| Product is missing, deleted, or cross-owner | `404 Error.ProductNotFound`     |
| Product is `SUSPENDED` or `ARCHIVED`        | `409 Error.ProductNotEditable`  |
| Slug is already reserved by another Product | `409 Error.ProductSlugConflict` |
| Media validation fails                      | `422 Error.ProductMediaInvalid` |
| Caller is not an eligible active Publisher  | Existing Publisher policy error |

### 5.4 List Products for management

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
        "name": "Daily Reward Collector",
        "slug": "daily-reward-collector",
        "shortDescription": "Collect daily rewards with basic Android actions.",
        "description": "An editable Product draft.",
        "usageInstructions": null,
        "media": [],
        "currentReleaseId": null,
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

### 5.5 Get Product detail for management

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

Successful response: `200 OK` with `ProductManagementDetailOutput`.

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
    "name": "Daily Reward Collector",
    "slug": "daily-reward-collector",
    "shortDescription": "Collect daily rewards with basic Android actions.",
    "description": "An editable Product draft.",
    "usageInstructions": null,
    "media": [
      {
        "id": "18ce7863-8a10-4be2-8f44-0b8a356ebf2a",
        "staticId": "d31977c8-d7ec-4a5d-8784-f9ef230a64ef",
        "type": "ICON",
        "sortOrder": 0,
        "altText": "Daily Reward Collector icon"
      }
    ],
    "offers": [
      {
        "id": "aa4e2b8c-e255-4d18-a8ec-a8606d3489e7",
        "type": "ONE_TIME",
        "priceAmount": "0",
        "durationDays": 7
      },
      {
        "id": "232b6676-988d-4a2a-9cbb-11633a390695",
        "type": "SUBSCRIPTION",
        "priceAmount": "250",
        "durationDays": 90
      }
    ],
    "currentReleaseId": null,
    "statusReason": null,
    "suspendedAt": null,
    "createdAt": "2026-10-19T08:00:00.000Z",
    "updatedAt": null
  },
  "statusCode": 200
}
```

The management detail response returns the current non-deleted Offers so the Publisher editor and Admin view can reload the newest state after an Offer mutation. Offer IDs are read-only; the strict Offer replacement request accepts only type, price, and duration. Product management lists do not preload Offers.

The management response does not expose audit user IDs, soft-delete fields, workflow content, Publisher private data, Submission snapshots, Static object keys, or Release artifacts.

Missing, deleted, and cross-owner Products return `404 Error.ProductNotFound`. Invalid UUIDs return `422` before the service runs.

### 5.6 Public Marketplace Product reads

This contract belongs to the Marketplace Product controller and reuses the approved Submission,
current Product Release, and Product Offer projections. It is separate from Product authoring.

Public Marketplace reads use separate routes from management reads:

```http
GET /api/v1/marketplace/products?page=1&limit=20&q=reward
GET /api/v1/marketplace/products/{productId}
```

The routes belong to `MarketplaceProductController`, use `@IsPublic()`, accept no Bearer token
requirement, and have no Permission record or scope. They must not reuse the permission-scoped
`GET /products` and `GET /products/:productId` controller methods.

Eligibility is identical for list and detail. A public Product must satisfy all of these conditions:

- Product is non-deleted and `ACTIVE`;
- owner Publisher and User remain publicly eligible and non-deleted;
- `currentReleaseId` resolves to a release belonging to the same Product;
- current release is `PUBLISHED`, not disabled, and references an `APPROVED` Submission;
- at least one current non-deleted `SUBSCRIPTION` Offer is eligible for acquisition; and
- the response is projected from that Submission's immutable Product snapshot, not mutable Product
  presentation fields.

Reusable public shapes:

```ts
type PublicProductPublisherOutput = {
  username: string;
  displayName: string;
};

type PublicProductReleaseOutput = {
  id: string; // UUID
  version: string;
  publishedAt: string;
  minimumLauncherVersion: string;
  minimumRuntimeVersion: string;
};

type PublicProductSummaryOutput = {
  id: string; // stable Product UUID
  name: string;
  slug: string;
  shortDescription: string;
  publisher: PublicProductPublisherOutput;
  currentRelease: PublicProductReleaseOutput;
  minimumSubscriptionOffer: {
    priceAmount: string;
    durationDays: number;
  };
};

type PublicProductDetailOutput = PublicProductSummaryOutput & {
  description: string;
  usageInstructions: string | null;
  offers: ProductOfferOutput[];
};
```

Presentation fields still come from the approved Submission behind `currentReleaseId`. `offers`
comes from the current non-deleted ProductOffer collection because commercial terms are managed
independently and are not part of Submission/Admin review. A Product without an eligible
subscription Offer fails public eligibility and is absent from both Marketplace list and detail,
even when it has a published Release in history.

`minimumSubscriptionOffer` considers only current non-deleted `SUBSCRIPTION` Offers and ignores
`ONE_TIME` Offers. It selects `priceAmount ASC`; equal prices use `durationDays ASC`, then Offer ID
`ASC`, so the returned duration always corresponds to the selected minimum-price row. Public
eligibility guarantees this projection is non-null. The list query must load it for the page in one
set-based query and must not execute one Offer query per Product.

List query:

```ts
type GetPublicProductListInput = {
  page?: number; // positive integer, default 1
  limit?: number; // positive integer, default 20
  q?: string; // trimmed, 1..100 when present; basic name/description search
};
```

List returns the global success envelope containing the existing pagination shape with
`PublicProductSummaryOutput[]`. Default order is current release `publishedAt DESC`, then Product ID
`DESC`. Unknown query parameters return `422`.

Detail uses strict UUID `productId` and returns `PublicProductDetailOutput`. Missing Products and
Products that fail any public eligibility condition both return `404 Error.ProductNotFound`; the
route must not reveal draft, suspended, archived, blocked-Publisher, rejected, failed-build, or
disabled-release state.

There is no standalone public or management Offer read endpoint. Product detail is the refresh
source after `PUT /products/:productId/offers`; Offer mutation responses contain only the shared
`SUCCESS_RESPONSE`.

Public media delivery is not part of this MVP API contract yet. These public responses must not
expose `staticId`, private Static URLs, or R2 object keys. Product media remains available in
Publisher/Admin management responses and immutable Submission snapshots until a separately reviewed
public media-delivery contract is added.

### 5.7 Publisher Product lifecycle

```http
PUT /api/v1/products/{productId}/lifecycle
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contract:

```text
method = PUT, path = /api/v1/products/:productId/lifecycle, scope = OWN  -> PUBLISHER
```

Strict discriminated request body:

```ts
type PublisherProductLifecycleInput =
  | {
      action: "ACTIVATE";
    }
  | {
      action: "ARCHIVE";
    }
  | {
      action: "REACTIVATE";
    };
```

Each union member is strict. Unknown actions or fields return `422`.

Examples:

```json
{
  "action": "ACTIVATE"
}
```

```json
{
  "action": "ARCHIVE"
}
```

```json
{
  "action": "REACTIVATE"
}
```

Action behavior:

| Action       | Allowed transition   | Idempotent state | Behavior                                                                                 |
| ------------ | -------------------- | ---------------- | ---------------------------------------------------------------------------------------- |
| `ACTIVATE`   | `DRAFT -> ACTIVE`    | `ACTIVE`         | Require eligible current Release and subscription Offer; record initial Offer timestamp. |
| `ARCHIVE`    | `ACTIVE -> ARCHIVED` | `ARCHIVED`       | Hide from Marketplace and block new purchases; retain customer access and all history.   |
| `REACTIVATE` | `ARCHIVED -> ACTIVE` | `ACTIVE`         | Recheck Release, Offer, and Publisher eligibility; consume no Offer-change slot.         |

`ACTIVATE` and `REACTIVATE` run in one transaction: lock the owned Product first, call
`ProductReleaseService.assertCurrentReleaseEligible` to lock/validate its current Release, then call
`ProductOfferService.assertActivationReady(productId)`. `ACTIVATE` additionally calls
`recordInitialPublicConfiguration` with the database transaction time. `ARCHIVE` retains
`currentReleaseId`, Offer rows, and Offer rate-limit history. All repository calls resolve
`PrismaService.getClient()` so they join the ambient transaction.

### 5.8 Admin Product moderation

```http
PUT /api/v1/products/{productId}/moderation
Authorization: Bearer <access-token>
Content-Type: application/json
```

Permission contract:

```text
method = PUT, path = /api/v1/products/:productId/moderation, scope = null -> ADMIN
```

```ts
type AdminProductModerationInput =
  | {
      action: "SUSPEND";
      reason: string; // trim, 1..2,000 characters
    }
  | {
      action: "REACTIVATE";
    };
```

`reason` is required only for `SUSPEND` and rejected for `REACTIVATE`. The Admin route never accepts
`ACTIVATE` or `ARCHIVE`.

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

| Action       | Allowed transition    | Idempotent state | Behavior                                                                      |
| ------------ | --------------------- | ---------------- | ----------------------------------------------------------------------------- |
| `SUSPEND`    | `ACTIVE -> SUSPENDED` | `SUSPENDED`      | Record the trimmed reason, Admin actor, and timestamp.                        |
| `REACTIVATE` | `SUSPENDED -> ACTIVE` | `ACTIVE`         | Recheck current Release and subscription Offer, then clear suspension fields. |

Publisher lifecycle permission cannot reach the moderation route. Admin moderation permission
cannot reach the Publisher lifecycle route. A Publisher therefore cannot override `SUSPENDED`, and
Admin moderation cannot override `ARCHIVED`.

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

| Condition                                                           | Result                                       |
| ------------------------------------------------------------------- | -------------------------------------------- |
| Product is missing, deleted, or outside an `OWN` caller's ownership | `404 Error.ProductNotFound`                  |
| Publisher lifecycle action targets `SUSPENDED`                      | `409 Error.ProductLifecycleUnavailable`      |
| Admin moderation action targets `ARCHIVED`                          | `409 Error.ProductModerationUnavailable`     |
| `ACTIVATE` requested for `ARCHIVED` Product                         | `409 Error.ProductCannotBeActivated`         |
| `ARCHIVE` requested for `DRAFT` Product                             | `409 Error.ProductCannotBeArchived`          |
| `SUSPEND` requested for `DRAFT` or `ARCHIVED`                       | `409 Error.ProductCannotBeSuspended`         |
| Publisher `REACTIVATE` requested for `DRAFT` Product                | `409 Error.ProductCannotBeReactivated`       |
| Admin `REACTIVATE` requested for `DRAFT` Product                    | `409 Error.ProductCannotBeReactivated`       |
| Activation/reactivation without an eligible current Release         | `409 Error.ProductReleaseUnavailable`        |
| Activation/reactivation without an eligible subscription Offer      | `409 Error.ProductSubscriptionOfferRequired` |
| Invalid UUID, action, reason, or body shape                         | `422` validation error                       |
| `OWN` caller is no longer an eligible active Publisher              | Existing Publisher policy error              |

There is no Product delete endpoint. Public reads are available only through the `@IsPublic()`
Marketplace projection routes in section 5.6.

### 5.9 Endpoint summary

| Method  | Path                                      | Access                      | Responsibility                                     |
| ------- | ----------------------------------------- | --------------------------- | -------------------------------------------------- |
| `POST`  | `/api/v1/products`                        | Authenticated Publisher     | Create one Product draft and return its ID.        |
| `PATCH` | `/api/v1/products/:productId`             | Publisher `OWN`             | Update mutable Product draft fields and media.     |
| `GET`   | `/api/v1/products`                        | `OWN` or unrestricted Admin | List Products for management.                      |
| `GET`   | `/api/v1/products/:productId`             | `OWN` or unrestricted Admin | Read Product management detail.                    |
| `GET`   | `/api/v1/marketplace/products`            | `@IsPublic()`               | List publicly eligible Marketplace Products.       |
| `GET`   | `/api/v1/marketplace/products/:productId` | `@IsPublic()`               | Read publicly eligible Marketplace Product detail. |
| `PUT`   | `/api/v1/products/:productId/lifecycle`   | Publisher `OWN`             | Activate, archive, or reactivate an owned Product. |
| `PUT`   | `/api/v1/products/:productId/moderation`  | Unrestricted Admin          | Suspend or reactivate a moderated Product.         |

The two Marketplace routes are part of the Product API contract but have no stored Permission row,
scope, or role assignment. `@IsPublic()` bypasses Bearer authentication and permission resolution.

### 5.10 Permission records and role assignment

Permission uniqueness follows the existing `(method, path, scope)` contract. The Product feature
requires the following permission records; `null` means unrestricted scope.

| Permission name              | Module  | Method  | Stored path                              | Scope  | Assigned roles          |
| ---------------------------- | ------- | ------- | ---------------------------------------- | ------ | ----------------------- |
| Create Product Draft         | Product | `POST`  | `/api/v1/products`                       | `null` | `PUBLISHER`             |
| Update Own Product Draft     | Product | `PATCH` | `/api/v1/products/:productId`            | `OWN`  | `PUBLISHER`             |
| List Own Products            | Product | `GET`   | `/api/v1/products`                       | `OWN`  | `CUSTOMER`, `PUBLISHER` |
| List All Products            | Product | `GET`   | `/api/v1/products`                       | `null` | `ADMIN`                 |
| Read Own Product             | Product | `GET`   | `/api/v1/products/:productId`            | `OWN`  | `CUSTOMER`, `PUBLISHER` |
| Read Any Product             | Product | `GET`   | `/api/v1/products/:productId`            | `null` | `ADMIN`                 |
| Manage Own Product Lifecycle | Product | `PUT`   | `/api/v1/products/:productId/lifecycle`  | `OWN`  | `PUBLISHER`             |
| Moderate Product             | Product | `PUT`   | `/api/v1/products/:productId/moderation` | `null` | `ADMIN`                 |

No Product permission is assigned implicitly because a user has a Publisher account. Access still
requires an active role containing the matching permission record.
Public Marketplace routes are intentionally absent from this table because `@IsPublic()` bypasses
Bearer authentication and permission resolution.

## 6. Authorization and ownership behavior

- Product create and update call `PublisherAccessPolicy.assertActivePublisher(userId)`. This
  requires an active User, active non-deleted Publisher account, active Publisher role, and
  acceptance of current Publisher Terms.
- Create resolves an active, non-deleted Project using both `projectId` and the resolved
  `publisherAccountId`. Missing, archived, soft-deleted, and cross-owner Projects are externally
  indistinguishable.
- Product ownership is always derived from that Project. Create/update never accept a client-selected
  `publisherAccountId`.
- Create/update validate all submitted Static IDs in one set-based query as completed, non-deleted
  image-pipeline records. Static is shared and has no Publisher ownership boundary.
- `OWN` list/detail reads resolve the caller's non-deleted Publisher account regardless of
  `ACTIVE`/`BLOCKED` status, following the existing Project read pattern. A blocked former Publisher
  may inspect owned Product history through Customer `OWN` read permission.
- Publisher lifecycle access requires an active Publisher and current Publisher Terms, filters by
  the resolved `publisherAccountId`, and permits `ACTIVATE`, `ARCHIVE`, and `REACTIVATE` only on
  their documented source states.
- Admin moderation access is unrestricted by ownership and permits only `SUSPEND` and `REACTIVATE`.
- The routes, permissions, and strict request schemas are separate; authorization does not depend on
  interpreting a mixed Publisher/Admin action after entering one shared route.
- Admin list/detail access is unrestricted and does not resolve the Admin's Publisher account.
- Repositories consume a typed Product access predicate. They do not inspect role names, access-token
  payload structure, Redis permission-cache contents, or raw permission records.
- Missing IDs and cross-owner `OWN` access return the same not-found response to prevent resource
  enumeration.
- `UserStatus.BANNED` remains denied by the existing authentication/authorization layer.

The authoritative Product route permissions and role assignments are defined in section 5.10.
Submission permissions are defined by the Submission feature spec. Permission
creation and role assignment use the existing permission APIs and must preserve every existing
role-permission ID.

## 7. Repository, transaction, and concurrency contract

- Product create locks or conditionally selects the owned Project and commits Product plus initial
  ProductMedia in one transaction. The unique `Product.projectId` constraint resolves concurrent
  create attempts.
- Product update validates media in one set-based query and updates scalar fields plus the supplied
  media replacement in one last-write-wins transaction.
- Submission never creates Product. It locks the existing Product and exact Project Draft using the
  documented Workflow Revision lock order, validates Product completeness, and freezes Product/media
  plus Workflow Revision and release request snapshots.
- The same Submission transaction creates the immutable Workflow Revision and Submission. A failure
  rolls back both without changing Product draft data. After commit, Submission publishes the scan
  message directly through the Kafka provider; no outbox record is created.
- Never trust caller-supplied Publisher IDs and never create or edit Product from Admin review.
- A Product is eligible for Submission only when it matches the locked Project/Publisher and is
  `DRAFT` or `ACTIVE`. `ARCHIVED` and `SUSPENDED` reject new Submission creation.
- Product lifecycle and moderation actions use atomic conditional updates on the expected source
  status. Activation/reactivation locks Product, its selected current ProductRelease, then current
  ProductOffer rows before changing Product status. Current-Release selection separately locks
  Product and Release before changing only the pointer.
- When a conditional mutation changes no row, make one authorization-safe read to distinguish
  idempotent completion, not found, and invalid lifecycle state without revealing cross-owner data.
- Do not use a Redis distributed lock. Database uniqueness and conditional writes are the durable
  concurrency boundaries.
- Do not query inside loops. Product list/media projections and Static validation are set-based.
- Do not perform object-storage, queue, Runtime, or other network calls inside Product transactions;
  Product only links already uploaded Static records.

## 8. Internal integration contracts

This section defines the Product contracts consumed by Submission and Product Release. It is not
implemented inside Product controllers.

The Submission module consumes an existing Product draft. After locking and authorizing the
Product, Project, and exact Project Draft, it calls a read-only snapshot contract inside the
Submission transaction:

```ts
getSnapshotForSubmission({
  productId,
  projectId,
  publisherAccountId,
}): Promise<{
  productSnapshot: ProductSubmissionSnapshot;
}>;
```

The method verifies:

- Product is non-deleted and belongs to the active submitting Publisher;
- Product references the selected locked Project and is `DRAFT` or `ACTIVE`;
- required publishing fields are complete and slug remains valid;
- every active ProductMedia reference remains an eligible completed Static image.

The snapshot contains Product identity, presentation fields, and ordered media metadata. It contains
no private Static object key or mutable URL. The enclosing transaction freezes this Product snapshot,
the immutable Workflow Revision, review metadata, and Release request. Submission creation does not
mutate Product draft fields or `currentReleaseId`.

The Product Release module owns release persistence. Its publication transaction creates
selectable Release history but never changes Product status or `currentReleaseId`. Its explicit
current-Release selection transaction owns only initial selection, update selection, and rollback:

```ts
selectCurrentRelease({
  productId,
  releaseId,
  actorId,
}): Promise<void>
```

That operation verifies that the release belongs to the same Product, references an approved
Submission, and is `PUBLISHED`, non-disabled, and package-complete. It sets only
`Product.currentReleaseId` plus Product audit fields. It never reads Offers, changes Product status,
or consumes an Offer sliding-window slot. An owner Publisher may select for `DRAFT`, `ACTIVE`, or
`ARCHIVED`; only Admin may select while `SUSPENDED`.

`ProductService` owns activation. It injects the existing `ProductReleaseService` and
`ProductOfferService` and calls `assertCurrentReleaseEligible` and `assertActivationReady` in the
ambient lifecycle transaction.
`DRAFT -> ACTIVE` records the initial public Offer timestamp. `ARCHIVED -> ACTIVE` and
`SUSPENDED -> ACTIVE` revalidate readiness but do not append a timestamp.

Admin approval changes Submission review state and schedules/allows the release pipeline. Successful
package publication still leaves Product status and current selection unchanged. Selection prepares
the pointer; Publisher activation is the explicit operation that makes a draft Product public.

Catalog, Offer, Entitlement, and Download Authorization modules must not infer availability from
Product status alone. Their queries must also evaluate the current release and their own domain
rules.

Product Release retains immutable older releases, lists releases for an owned Product, and allows an
authorized Publisher or Admin to point `currentReleaseId` to an eligible previously published,
non-disabled release. Selecting an old approved release is a pointer change, not a new Submission,
review, rebuild, Offer rate-limit event, or mutation of that ProductRelease. Exact endpoints,
permissions, audit behavior, and concurrency rules are defined by the Product Release specification.

## 9. Stable failure semantics

| Condition                                                             | Result                                       |
| --------------------------------------------------------------------- | -------------------------------------------- |
| Missing, soft-deleted, or cross-owner Product                         | `404 Error.ProductNotFound`                  |
| Missing, archived, soft-deleted, or cross-owner Project during create | `404 Error.ProjectNotFound`                  |
| Duplicate Product slug                                                | `409 Error.ProductSlugConflict`              |
| Invalid, unavailable, or unprocessed Product media                    | `422 Error.ProductMediaInvalid`              |
| Archived Product selected for update/submission                       | `409 Error.ProductArchived`                  |
| Suspended Product selected for update/submission                      | `409 Error.ProductSuspended`                 |
| Publisher lifecycle action targets `SUSPENDED`                        | `409 Error.ProductLifecycleUnavailable`      |
| Admin moderation action targets `ARCHIVED`                            | `409 Error.ProductModerationUnavailable`     |
| Activation without an eligible current Release                        | `409 Error.ProductReleaseUnavailable`        |
| Activation without an eligible subscription Offer                     | `409 Error.ProductSubscriptionOfferRequired` |
| Archive requested for `DRAFT` Product                                 | `409 Error.ProductCannotBeArchived`          |
| Suspend requested for non-`ACTIVE` Product                            | `409 Error.ProductCannotBeSuspended`         |
| Reactivate requested from `DRAFT`                                     | `409 Error.ProductCannotBeReactivated`       |
| Reactivation without an eligible current release                      | `409 Error.ProductReleaseUnavailable`        |
| Reactivation without an eligible subscription Offer                   | `409 Error.ProductSubscriptionOfferRequired` |
| Invalid UUID, status filter, body shape, or reason                    | `422` from the global Zod validation pipe    |
| Inactive/blocked/unqualified Publisher                                | Existing stable Publisher policy error       |
| Missing route permission or insufficient scope                        | `403` from the existing authorization layer  |

Safe error responses may include a current lifecycle status only after ownership/unrestricted access
has been established. They must not include Project content, Publisher private data, Submission
snapshots, package identifiers, or release secrets.

## 10. Privacy, logging, and operational behavior

- Do not add routine success logs for Product create, draft update, list, detail, or lifecycle
  actions. Centralized request logging and audit columns cover normal operations.
- Centralized request/error logging covers expected domain failures.
- Admin suspension/reactivation may log Product ID, actor ID, old/new status, and a stable action
  code. Do not log the free-text reason; store it only in Product and the platform Audit Log.
- Never log Project workflow content, Runtime IR, Publisher Terms payloads, access tokens, private
  profile fields, or release/package content.
- Product mutations set the appropriate audit actor and timestamp fields.
- Product lifecycle and current-Release selection emit no external events.

## 11. Acceptance criteria

The Product authoring feature is conformant when the following behaviors are demonstrated. Current
verification is manual; automated Product tests are intentionally deferred to prioritize feature
implementation.

| Area             | Required behavior                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| Identity         | Exactly one Product is created by its Publisher for a Project before Submission.                                    |
| Ownership        | Product ownership always equals Project ownership and cannot be supplied or changed by clients.                     |
| Draft authoring  | Product text and ordered Static-backed media save together with MVP last-write-wins behavior.                       |
| Create conflict  | Exactly one create wins per Project; repeated and concurrent losers receive a terminal conflict.                    |
| Publisher policy | Only a currently qualified owner Publisher can activate, archive, or reactivate through its lifecycle route.        |
| Read access      | Owner `OWN` and Admin unrestricted reads are isolated; cross-owner access is indistinguishable from not found.      |
| Lifecycle        | Publisher and Admin routes enforce their separate transitions; neither role can override the other's hidden state.  |
| Existing access  | `ARCHIVED` blocks discovery/new purchase but preserves eligible access for existing unexpired Entitlements.         |
| Project boundary | Archived Project does not erase Product history, but cannot create a new Product or support another Submission.     |
| Data exposure    | Management responses expose draft fields and Static IDs, never object keys, private URLs, or workflow content.      |
| Performance      | Lists and relation projections are set-based, paginated, deterministically ordered, and contain no per-row queries. |
| Security         | No client can select another Publisher, enumerate Products, or mutate lifecycle via PATCH.                          |
| Operations       | Expected mutations use database constraints/conditional writes and add no Redis lock or storage/network call.       |

## 12. Deferred features

- Category, Tag, ProductTag, featured placement, required-app classification, and connector taxonomy.
- General Asset-backed Product media, videos, and large-file/presigned upload flows.
- Public Marketplace delivery of approved Product images.
- Presigned upload/download URLs.
- Submission, automated scan, review, and Workflow Revision orchestration.
- Submission snapshot integration and Product publishing-completeness validation.
- Product Release persistence, `currentReleaseId` foreign key, release history/current-release
  selection, package creation, encryption, and integrity data.
- Advanced catalog ranking, recommendations, and full-text search infrastructure.
- Product Offer, acquisition, entitlement, order, payment, download authorization, and execution
  authorization.
- Product transfer between Publishers or Projects.
- Publisher team ownership.
- Product hard-delete, restoration, or reuse of a Project after archival.
