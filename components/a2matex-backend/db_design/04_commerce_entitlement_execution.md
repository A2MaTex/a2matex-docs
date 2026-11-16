# 04 — Offer, Order, Entitlement và Execution Authorization

## 1. Trách nhiệm nhóm

Nhóm này quản lý quyền sử dụng Product của customer:

```text
ProductOffer
→ zero-value hoặc balance Order
→ Entitlement
→ ExecutionLease
→ Runtime validation
→ ExecutionReport
```

MVP dùng `Profile.balance` làm system currency và vẫn tạo Order/OrderItem cho acquisition giá 0.
Direct payment, multi-currency và PaymentTransaction cho purchase Product được defer đến một feature
spec riêng. Admin manual grant vẫn thuộc Entitlement Module.

## 2. Product Offer

### Trách nhiệm

`ProductOffer` mô tả các lựa chọn one-time/subscription hiện tại của một Product. Publisher thay thế
toàn bộ collection bằng một API; Backend match current row theo `(productId, type, durationDays)`,
update price in-place, insert combination mới và soft-delete combination bị
omit. Offer không có review/status riêng và không nằm trong Submission snapshot. Contract đầy đủ:
[Product Offer Module Feature Spec](../commerce_entitlement_execution/product_offer.md).

```sql
CREATE TYPE "ProductOfferType" AS ENUM (
    'ONE_TIME',
    'SUBSCRIPTION'
);

ALTER TABLE "Product"
ADD COLUMN "offerChangeTimestamps" TIMESTAMPTZ[] NOT NULL DEFAULT '{}';

CREATE TABLE "ProductOffer" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "productId" UUID NOT NULL REFERENCES "Product" (id) ON DELETE NO ACTION,
    type "ProductOfferType" NOT NULL,
    "priceAmount" BIGINT NOT NULL,
    "durationDays" INTEGER,

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,
    "createdById" UUID NOT NULL REFERENCES "User" (id) ON DELETE NO ACTION,
    "updatedById" UUID REFERENCES "User" (id) ON DELETE NO ACTION,
    "deletedById" UUID REFERENCES "User" (id) ON DELETE NO ACTION
);

CREATE INDEX "ProductOffer_productId_idx"
ON "ProductOffer" ("productId")
WHERE "deletedAt" IS NULL;
```

`priceAmount` là BIGINT cùng unit với `Profile.balance` và được serialize thành decimal string trên
API. `ONE_TIME` cho phép price bằng 0, duration hữu hạn hoặc permanent (`null`). `SUBSCRIPTION` yêu
cầu price dương và duration hữu hạn. Backend validate các rule này, duplicate natural key,
soft-delete pairing và sliding window; database chỉ giữ identity/foreign key/index quan trọng.

`Product.offerChangeTimestamps` là bounded internal array. Product Offer service chỉ prune và append
khi collection của Product `ACTIVE` thay đổi thật sự. Cutoff dùng database transaction time trừ
`PRODUCT_OFFER_CHANGE_WINDOW_HOURS`; request bị reject với
`Error.ProductOfferChangeRateLimitExceeded` khi số timestamp còn lại đã đạt
`PRODUCT_OFFER_CHANGE_MAX_COUNT`. Draft edit, no-op, request invalid/unauthorized và transaction fail
không consume slot. Không expose timestamps, remaining slots hoặc next allowed time qua API.

Product `DRAFT` có thể lưu collection rỗng. Current-Release selection chỉ set pointer và không đọc
Offer. Publisher lifecycle `ACTIVATE` inject existing `ProductOfferService`, yêu cầu ít nhất một
current `SUBSCRIPTION` Offer và ghi timestamp đầu tiên trong cùng transaction chuyển Product sang
`ACTIVE`. Publisher/Admin reactivation và đổi current Release không consume slot. Khi Product đã
`ACTIVE`, replacement payload phải giữ ít nhất một `SUBSCRIPTION`; request làm mất Offer subscription
cuối cùng bị reject toàn bộ. `SUSPENDED`, `ARCHIVED` và deleted Product không cho phép replace Offer.

Mọi giới hạn lấy từ validated environment configuration:

```text
PRODUCT_OFFER_MAX_COUNT
PRODUCT_OFFER_MAX_DURATION_DAYS
PRODUCT_OFFER_CHANGE_MAX_COUNT
PRODUCT_OFFER_CHANGE_WINDOW_HOURS
PRODUCT_OFFER_MAX_PRICE_AMOUNT
```

Offer đã soft-delete là historical data immutable: không restore, không update và không reuse ID.
Nếu cùng type/duration xuất hiện lại, Backend insert row mới.

Không có Offer GET API riêng. Management và public Product detail preload toàn bộ current
non-deleted Offer; public Product list chỉ preload minimum-priced `SUBSCRIPTION` Offer gồm price và
duration, hoàn toàn bỏ qua `ONE_TIME`. Public eligibility yêu cầu projection này tồn tại nên list
item không trả minimum price nullable. `PUT /products/:productId/offers` chỉ trả
`SUCCESS_RESPONSE`; frontend reload Product detail để lấy state và generated IDs mới nhất.

## 3. Order

### Trách nhiệm

`Order` lưu transaction dùng system balance. `OrderItem` snapshot Offer để lịch sử không thay đổi
khi Publisher sửa price hoặc collection. Order locks Product trước selected ProductOffer, rồi lock
customer balance/Entitlement state theo một thứ tự nhất quán. Purchase transaction re-read current
Offer và không tin price/duration do client gửi.

```sql
CREATE TYPE "OrderStatus" AS ENUM (
    'PENDING',
    'PAID',
    'CANCELLED',
    'FAILED'
);

CREATE TABLE "Order" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "customerId" UUID NOT NULL REFERENCES "User" (id),
    status "OrderStatus" NOT NULL DEFAULT 'PENDING',
    "totalAmount" BIGINT NOT NULL,
    "paidAt" TIMESTAMPTZ,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    CONSTRAINT "Order_paidAt_check" CHECK (
        status <> 'PAID' OR "paidAt" IS NOT NULL
    )
);

CREATE TABLE "OrderItem" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "orderId" UUID NOT NULL REFERENCES "Order" (id),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "productOfferId" UUID NOT NULL REFERENCES "ProductOffer" (id),
    "unitPriceAmount" BIGINT NOT NULL,

    -- Mock JSON: immutable commercial snapshot.
    -- {
    --   "schemaVersion": "1.0",
    --   "offerType": "ONE_TIME",
    --   "priceAmount": "0",
    --   "durationDays": 30
    -- }
    "offerSnapshot" JSONB NOT NULL,

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "OrderItem_order_product_key" UNIQUE ("orderId", "productId"),
    CONSTRAINT "OrderItem_offerSnapshot_object_check"
        CHECK (jsonb_typeof("offerSnapshot") = 'object')
);
```

Acquisition giá 0 vẫn tạo zero-value Order và OrderItem. Product purchase không tạo
PaymentTransaction trong MVP; TopUp/SePay chỉ nạp `Profile.balance` trước khi Order dùng balance.

## 4. Entitlement

### Trách nhiệm

`Entitlement` là current access state lâu dài của một User đối với một Product. Immutable commercial terms nằm trong từng OrderItem; Entitlement giữ period và source mới nhất để authorize access.
Entitlement không giới hạn machine, device hoặc concurrent execution.

```sql
CREATE TYPE "EntitlementStatus" AS ENUM (
    'ACTIVE',
    'EXPIRED',
    'REVOKED'
);

CREATE TYPE "EntitlementGrantSource" AS ENUM (
    'ORDER',
    'ADMIN_GRANT'
);

CREATE TABLE "Entitlement" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "customerId" UUID NOT NULL REFERENCES "User" (id),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "latestProductOfferId" UUID REFERENCES "ProductOffer" (id),
    "latestOrderItemId" UUID REFERENCES "OrderItem" (id),
    status "EntitlementStatus" NOT NULL DEFAULT 'ACTIVE',
    "grantSource" "EntitlementGrantSource" NOT NULL,
    "startsAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "expiresAt" TIMESTAMPTZ,
    "oneTimePurchasedAt" TIMESTAMPTZ,

    -- Mock JSON: current effective usage rights; OrderItem keeps immutable purchase terms.
    -- {
    --   "schemaVersion": "1.0",
    --   "releaseAccess": "CURRENT_AND_SUPPORTED",
    --   "allowUpdates": true
    -- }
    "grantSnapshot" JSONB NOT NULL,

    "grantedById" UUID REFERENCES "User" (id),
    "revokedAt" TIMESTAMPTZ,
    "revokedById" UUID REFERENCES "User" (id),
    "revokedReason" TEXT,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,

    CONSTRAINT "Entitlement_period_check"
        CHECK ("expiresAt" IS NULL OR "expiresAt" > "startsAt"),
    CONSTRAINT "Entitlement_grantSnapshot_object_check"
        CHECK (jsonb_typeof("grantSnapshot") = 'object'),
    CONSTRAINT "Entitlement_revoke_fields_check" CHECK (
        status <> 'REVOKED'
        OR ("revokedAt" IS NOT NULL AND "revokedById" IS NOT NULL AND "revokedReason" IS NOT NULL)
    ),
    CONSTRAINT "Entitlement_customer_product_key"
        UNIQUE ("customerId", "productId")
);
```

Một customer có đúng một durable Entitlement row cho mỗi Product. Order mới bị block nếu
`expiresAt IS NULL` hoặc `expiresAt > transactionNow`; status stale không thay thế time check. Khi
access đã expire, customer có thể mua `SUBSCRIPTION` mới và period bắt đầu từ transaction time, không
stack/extend period cũ. `oneTimePurchasedAt` được set ở lần `ONE_TIME` thành công đầu tiên và không
bao giờ clear, kể cả khi access expire. Mỗi customer chỉ được mua `ONE_TIME` một lần cho mỗi Product,
không phụ thuộc Offer ID. Không có stacking, early renewal, upgrade, downgrade, proration hoặc
remaining-value conversion. Order owns balance deduction, zero-value Order/OrderItem và atomic
Entitlement create/reactivation.

Order mới chỉ được tạo khi Product `ACTIVE`. `ARCHIVED` là Publisher unlisting nên block Marketplace
discovery và Order mới nhưng không revoke existing unexpired Entitlement; download/execution vẫn
được phép nếu current Release còn eligible. `SUSPENDED` là Admin moderation control và block cả
download/execution cho đến khi Admin reactivation thành công.

## 5. Execution Lease

### Trách nhiệm

`ExecutionLease` là authorization tạm thời cho một production execution của
customer. Backend không gắn lease với installation/device và không giới hạn số
lease đồng thời. Mỗi row là một lease family; Backend rotate signed lease ID và
sequence trên cùng row khi renew. Signed token, key envelope và plaintext key
không được lưu trong PostgreSQL.

Trong signed claims, row `id` là `leaseFamilyId`, còn `currentLeaseId` là
`leaseId` hiện tại.

```sql
CREATE TYPE "ExecutionLeaseStatus" AS ENUM (
    'ACTIVE',
    'EXPIRED',
    'REVOKED',
    'COMPLETED'
);

CREATE TABLE "ExecutionLease" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "executionId" UUID NOT NULL UNIQUE,
    "entitlementId" UUID NOT NULL REFERENCES "Entitlement" (id),
    "customerId" UUID NOT NULL REFERENCES "User" (id),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "productReleaseId" UUID NOT NULL REFERENCES "ProductRelease" (id),
    "packageHash" TEXT NOT NULL,
    "runtimeSessionId" UUID NOT NULL,
    "runtimeKeyThumbprint" TEXT NOT NULL,
    "currentLeaseId" UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
    sequence INTEGER NOT NULL DEFAULT 1 CHECK (sequence > 0),
    status "ExecutionLeaseStatus" NOT NULL DEFAULT 'ACTIVE',

    -- Mock JSON: authorization decision snapshot; never contains keys/tokens.
    -- {
    --   "schemaVersion": "1.0",
    --   "entitlementStatus": "ACTIVE",
    --   "productStatus": "ACTIVE",
    --   "releaseStatus": "PUBLISHED",
    --   "packageHash": "sha256:..."
    -- }
    "grantSnapshot" JSONB NOT NULL,

    "issuedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "notBefore" TIMESTAMPTZ NOT NULL,
    "refreshAfter" TIMESTAMPTZ NOT NULL,
    "expiresAt" TIMESTAMPTZ NOT NULL,
    "renewedAt" TIMESTAMPTZ,
    "revokedAt" TIMESTAMPTZ,
    "revokedReason" TEXT,
    "completedAt" TIMESTAMPTZ,
    "updatedAt" TIMESTAMPTZ,

    CONSTRAINT "ExecutionLease_period_check" CHECK (
        "issuedAt" <= "notBefore"
        AND "notBefore" < "refreshAfter"
        AND "refreshAfter" < "expiresAt"
    ),
    CONSTRAINT "ExecutionLease_grantSnapshot_object_check"
        CHECK (jsonb_typeof("grantSnapshot") = 'object'),
    CONSTRAINT "ExecutionLease_revoked_fields_check" CHECK (
        status <> 'REVOKED'
        OR ("revokedAt" IS NOT NULL AND "revokedReason" IS NOT NULL)
    )
);

CREATE INDEX "ExecutionLease_active_entitlement_idx"
ON "ExecutionLease" ("entitlementId", "expiresAt")
WHERE status = 'ACTIVE';

```

Issue/renew phải update `currentLeaseId`, `sequence`, `refreshAfter` và
`expiresAt` atomically; renew dùng row lock/idempotency để không cấp hai sequence
khác nhau cho cùng request. Backend signs claims from this row and wraps the
release content key for `runtimeKeyThumbprint`. Runtime verifies locally;
Launcher never receives plaintext workflow key. Contract đầy đủ:
[Production Execution Authorization v1](../../../specifications/production_execution_authorization_v1.md).

## 6. Execution Report

### Trách nhiệm

Backend chỉ nhận cloud run summary. Detailed node state/history thuộc Runtime Agent SQLite.

```sql
CREATE TYPE "ExecutionOutcome" AS ENUM (
    'SUCCEEDED',
    'FAILED',
    'CANCELLED',
    'INTERRUPTED'
);

CREATE TABLE "ExecutionReport" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "executionId" UUID NOT NULL UNIQUE,
    "executionLeaseId" UUID NOT NULL REFERENCES "ExecutionLease" (id),
    "runGroupId" VARCHAR(255) NOT NULL,
    outcome "ExecutionOutcome" NOT NULL,

    -- Mock JSON: aggregate only, no raw credential, ADB output or decrypted graph.
    -- {
    --   "schemaVersion": "1.0",
    --   "totalDevices": 3,
    --   "succeededDevices": 2,
    --   "failedDevices": 1,
    --   "cancelledDevices": 0,
    --   "durationMs": 125000,
    --   "failureCodes": ["DEVICE_OFFLINE"]
    -- }
    summary JSONB NOT NULL,

    "startedAt" TIMESTAMPTZ NOT NULL,
    "finishedAt" TIMESTAMPTZ NOT NULL,
    "receivedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "ExecutionReport_time_check" CHECK ("finishedAt" >= "startedAt"),
    CONSTRAINT "ExecutionReport_summary_object_check"
        CHECK (jsonb_typeof(summary) = 'object')
);
```

Unique `executionId` giúp endpoint report idempotent.

## 7. Product Report và moderation

### Trách nhiệm

MVP chỉ cần customer gửi report và admin resolve với reason.

```sql
CREATE TYPE "ProductReportStatus" AS ENUM (
    'OPEN',
    'RESOLVED',
    'DISMISSED'
);

CREATE TABLE "ProductReport" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "reporterUserId" UUID NOT NULL REFERENCES "User" (id),
    category VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    status "ProductReportStatus" NOT NULL DEFAULT 'OPEN',

    -- Mock JSON: references only to already uploaded evidence assets.
    -- {
    --   "schemaVersion": "1.0",
    --   "assetIds": ["uuid"],
    --   "releaseId": "uuid"
    -- }
    evidence JSONB NOT NULL DEFAULT '{}'::JSONB,

    "resolutionReason" TEXT,
    "resolvedAt" TIMESTAMPTZ,
    "resolvedById" UUID REFERENCES "User" (id),
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,

    CONSTRAINT "ProductReport_evidence_object_check"
        CHECK (jsonb_typeof(evidence) = 'object'),
    CONSTRAINT "ProductReport_resolution_check" CHECK (
        status = 'OPEN'
        OR ("resolutionReason" IS NOT NULL AND "resolvedAt" IS NOT NULL AND "resolvedById" IS NOT NULL)
    )
);

CREATE INDEX "ProductReport_status_createdAt_idx"
ON "ProductReport" (status, "createdAt");
```

## 8. Quan hệ chính

```text
Product
├── ProductOffer[]
├── OrderItem[] ── Order
├── Entitlement[]
│   └── ExecutionLease[] ── ProductRelease
│       └── ExecutionReport
└── ProductReport[]
```
