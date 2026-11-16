# 03 — Product, Submission, Review và ProductRelease

## 1. Trách nhiệm nhóm

Nhóm này quản lý hành trình publish trên Marketplace:

```text
Saved Project
→ Web submission preparation
→ editable Product + ProductMedia
→ immutable WorkflowRevision
→ Submission
→ automated scan
→ admin review
→ immutable ProductRelease
→ public catalog
```

`Product` là aggregate Publisher chỉnh sửa trước khi submit: identity ổn định, thông tin hiển thị và
con trỏ tới release đang publish. `ProductMedia` chỉ liên kết ảnh đã upload qua Static/R2.
`Submission`, `WorkflowRevision` và technical content của `ProductRelease` là immutable snapshot.

Category, Tag, ProductTag, required-app classification và connector taxonomy không nằm trong MVP.
Các node hiện tại chỉ cung cấp Android action cơ bản nên chưa có taxonomy ổn định mang giá trị tìm
kiếm. Khi catalog lớn hơn, taxonomy chỉ được thêm từ usage thực tế và một feature spec riêng.

## 2. Product aggregate và media

### Trách nhiệm

`Product` thuộc một Publisher, liên kết đúng một Studio Project, và tồn tại xuyên suốt nhiều
Submission/ProductRelease. Publisher có thể lưu draft chưa đầy đủ. Submission lock Product hiện tại,
validate đầy đủ field publish bắt buộc, rồi snapshot chính xác row đã lock.

```sql
CREATE TYPE "ProductStatus" AS ENUM (
    'DRAFT',
    'ACTIVE',
    'SUSPENDED',
    'ARCHIVED'
);

CREATE TABLE "Product" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "publisherAccountId" UUID NOT NULL REFERENCES "PublisherAccount" (id),
    "projectId" UUID NOT NULL UNIQUE REFERENCES "Project" (id),
    status "ProductStatus" NOT NULL DEFAULT 'DRAFT',
    name VARCHAR(255),
    slug VARCHAR(255),
    "shortDescription" VARCHAR(500),
    description TEXT,
    "usageInstructions" TEXT,
    "statusReason" TEXT,
    "suspendedAt" TIMESTAMPTZ,
    "suspendedById" UUID REFERENCES "User" (id),
    "currentReleaseId" UUID,
    "offerChangeTimestamps" TIMESTAMPTZ[] NOT NULL DEFAULT '{}',
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,
    "createdById" UUID NOT NULL REFERENCES "User" (id),
    "updatedById" UUID REFERENCES "User" (id),

    CONSTRAINT "Product_suspended_fields_check" CHECK (
        status <> 'SUSPENDED'
        OR ("statusReason" IS NOT NULL AND "suspendedAt" IS NOT NULL AND "suspendedById" IS NOT NULL)
    )
);

CREATE INDEX "Product_publisher_status_idx"
ON "Product" ("publisherAccountId", status)
WHERE "deletedAt" IS NULL;

CREATE UNIQUE INDEX "Product_slug_idx"
ON "Product" (lower(slug))
WHERE "deletedAt" IS NULL AND slug IS NOT NULL;

CREATE TYPE "ProductMediaType" AS ENUM (
    'ICON',
    'COVER',
    'GALLERY'
);

CREATE TABLE "ProductMedia" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "staticId" UUID NOT NULL REFERENCES "Static" (id),
    type "ProductMediaType" NOT NULL,
    "sortOrder" INTEGER NOT NULL DEFAULT 0,
    "altText" VARCHAR(500),
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ
);

CREATE UNIQUE INDEX "ProductMedia_product_static_idx"
ON "ProductMedia" ("productId", "staticId")
WHERE "deletedAt" IS NULL;

CREATE UNIQUE INDEX "ProductMedia_single_icon_idx"
ON "ProductMedia" ("productId")
WHERE "deletedAt" IS NULL AND type = 'ICON';

CREATE UNIQUE INDEX "ProductMedia_single_cover_idx"
ON "ProductMedia" ("productId")
WHERE "deletedAt" IS NULL AND type = 'COVER';
```

Draft Product mutation có thể thay đổi ProductMedia, nhưng không xóa Static/R2 object đang được một
Submission/ProductRelease snapshot tham chiếu. Marketplace không đọc trực tiếp mutable Product row;
nó đọc Product snapshot của current published release.

Static là storage metadata dùng chung và không có Product/Publisher ownership. ProductMedia có thể
tham chiếu bất kỳ Static image nào chưa bị xóa và đã hoàn tất image pipeline; request chỉ nhận
`staticId`, không nhận object key, private URL hoặc R2 key.

`offerChangeTimestamps` là bounded internal array do Product Offer module quản lý để enforce sliding
window cho thay đổi commercial collection của Product `ACTIVE`. Product create/update, Submission và
Review và current-Release selection không được ghi hoặc expose field này. Publisher lifecycle
`ACTIVATE` ghi entry đầu tiên thông qua `ProductOfferService`; Publisher/Admin reactivation và các
lần đổi Release không ghi thêm. Product Offer không nằm trong Submission/ProductRelease snapshot.

Product lifecycle được tách theo role:

```text
Publisher OWN /products/:productId/lifecycle
  ACTIVATE:   DRAFT -> ACTIVE
  ARCHIVE:    ACTIVE -> ARCHIVED
  REACTIVATE: ARCHIVED -> ACTIVE

Admin /products/:productId/moderation
  SUSPEND:    ACTIVE -> SUSPENDED
  REACTIVATE: SUSPENDED -> ACTIVE
```

`ARCHIVED` là Publisher unlisting có thể reverse: Product không xuất hiện trên Marketplace và không
nhận Order mới, nhưng giữ `currentReleaseId`, Offer, Release và existing Entitlement access. Publisher
không được thay đổi `SUSPENDED`; Admin moderation không được thay đổi `ARCHIVED`.

## 3. Submission

### Trách nhiệm

`Submission` là request review immutable về content snapshot. Status thay đổi theo review, nhưng các JSON snapshot và `workflowRevisionId` không được ghi đè.

```sql
CREATE TYPE "SubmissionStatus" AS ENUM (
    'PENDING_SCAN',
    'READY_FOR_REVIEW',
    'CHANGES_REQUESTED',
    'APPROVED',
    'REJECTED'
);

CREATE TABLE "Submission" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "workflowRevisionId" UUID NOT NULL UNIQUE REFERENCES "WorkflowRevision" (id),
    "previousSubmissionId" UUID REFERENCES "Submission" (id),
    number INTEGER NOT NULL CHECK (number > 0),
    status "SubmissionStatus" NOT NULL DEFAULT 'PENDING_SCAN',

    -- Mock JSON: complete Product working copy exactly as publisher confirmed at submit time.
    -- {
    --   "schemaVersion": "1.0",
    --   "productId": "uuid",
    --   "publisherAccountId": "uuid",
    --   "projectId": "uuid",
    --   "name": "Daily Reward Collector",
    --   "slug": "daily-reward-collector",
    --   "shortDescription": "Collect daily rewards",
    --   "description": "...",
    --   "usageInstructions": null,
    --   "media": [
    --     {"id":"uuid","staticId":"uuid","type":"ICON","sortOrder":0,"altText":"Product icon"}
    --   ]
    -- }
    "productSnapshot" JSONB NOT NULL,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "version": "1.0.0",
    --   "changelog": "Initial release",
    --   "minimumLauncherVersion": "1.0.0",
    --   "minimumRuntimeVersion": "1.0.0"
    -- }
    "releaseRequest" JSONB NOT NULL,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "purpose": "Collect daily reward",
    --   "testInstructions": "Use the provided test account",
    --   "knownLimitations": ["English UI only"]
    -- }
    "reviewContext" JSONB NOT NULL,

    "submittedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "submittedById" UUID NOT NULL REFERENCES "User" (id),
    "updatedAt" TIMESTAMPTZ,

    CONSTRAINT "Submission_product_number_key" UNIQUE ("productId", number),
    CONSTRAINT "Submission_json_objects_check" CHECK (
        jsonb_typeof("productSnapshot") = 'object'
        AND jsonb_typeof("releaseRequest") = 'object'
        AND jsonb_typeof("reviewContext") = 'object'
    )
);

CREATE INDEX "Submission_status_submittedAt_idx"
ON "Submission" (status, "submittedAt");
```

Một transaction submit nên:

1. Resolve Publisher và lock Product hiện có, Project, rồi ProjectDraft exact version theo lock order.
2. Validate Product publish fields và toàn bộ Static image refs từ Product row đã lock.
3. Revalidate graph/IR/assets và create WorkflowRevision immutable.
4. Snapshot Product fields + ordered media, release request và review context; không update mutable Product.
5. Create Submission.
6. Commit transaction, sau đó publish scan message trực tiếp qua Kafka provider. Không tạo outbox
   record. Nếu publish fail, retry cùng request và cùng Idempotency-Key sẽ republish chính
   `(submissionId, scanAttempt)` đã commit thay vì tạo Submission khác.

## 4. Automated Scan

### Trách nhiệm

`SubmissionScan` lưu một lần scanner chạy. `SubmissionScanFinding` lưu finding có thể filter/query mà không phải parse một JSON result lớn.

```sql
CREATE TYPE "ScanStatus" AS ENUM (
    'QUEUED',
    'RUNNING',
    'PASSED',
    'FAILED',
    'ERROR'
);

CREATE TYPE "FindingSeverity" AS ENUM (
    'INFO',
    'WARNING',
    'ERROR'
);

CREATE TABLE "SubmissionScan" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "submissionId" UUID NOT NULL REFERENCES "Submission" (id),
    attempt INTEGER NOT NULL DEFAULT 1 CHECK (attempt > 0),
    status "ScanStatus" NOT NULL DEFAULT 'QUEUED',
    "scannerVersion" VARCHAR(100) NOT NULL,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "totalRules": 20,
    --   "passed": 18,
    --   "warnings": 1,
    --   "errors": 1
    -- }
    summary JSONB NOT NULL DEFAULT '{}'::JSONB,

    "startedAt" TIMESTAMPTZ,
    "finishedAt" TIMESTAMPTZ,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "SubmissionScan_submission_attempt_key"
        UNIQUE ("submissionId", attempt),
    CONSTRAINT "SubmissionScan_summary_object_check"
        CHECK (jsonb_typeof(summary) = 'object')
);

CREATE TABLE "SubmissionScanFinding" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "submissionScanId" UUID NOT NULL REFERENCES "SubmissionScan" (id),
    code VARCHAR(255) NOT NULL,
    severity "FindingSeverity" NOT NULL,
    message TEXT NOT NULL,
    "nodeId" VARCHAR(255),
    "assetId" UUID REFERENCES "Asset" (id),

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "jsonPath": "$.nodes[3].config.timeoutMs",
    --   "expected": "<= 60000",
    --   "actual": 120000
    -- }
    details JSONB NOT NULL DEFAULT '{}'::JSONB,

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "SubmissionScanFinding_details_object_check"
        CHECK (jsonb_typeof(details) = 'object')
);
```

## 5. Admin Review

### Trách nhiệm

`SubmissionReview` là append-only history của admin decisions/feedback. `Submission.status` giữ current state để query queue nhanh.

```sql
CREATE TYPE "ReviewDecision" AS ENUM (
    'REQUEST_CHANGES',
    'APPROVE',
    'REJECT'
);

CREATE TABLE "SubmissionReview" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "submissionId" UUID NOT NULL REFERENCES "Submission" (id),
    decision "ReviewDecision" NOT NULL,
    reason TEXT,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "items": [
    --     {"code":"PRODUCT_PRESENTATION_ACCURACY","result":"PASS","comment":null},
    --     {"code":"CAPABILITY_JUSTIFIED","result":"CHANGE_REQUIRED","comment":"Explain HTTP access"}
    --   ]
    -- }
    checklist JSONB NOT NULL DEFAULT '{}'::JSONB,

    "reviewedById" UUID NOT NULL REFERENCES "User" (id),
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "SubmissionReview_checklist_object_check"
        CHECK (jsonb_typeof(checklist) = 'object'),
    CONSTRAINT "SubmissionReview_reason_check" CHECK (
        decision = 'APPROVE' OR reason IS NOT NULL
    )
);
```

## 6. ProductRelease

### Trách nhiệm

`ProductRelease` lưu immutable technical artifact metadata và trỏ tới approved Submission. Khi
`PUBLISHED`, package fields không được update; disable chỉ thay status/reason. Public Marketplace
đọc Product presentation snapshot từ Submission của `Product.currentReleaseId`, không đọc trực tiếp
mutable Product row.

```sql
CREATE TYPE "ProductReleaseStatus" AS ENUM (
    'PENDING_BUILD',
    'BUILDING',
    'PUBLISHED',
    'BUILD_FAILED',
    'DISABLED'
);

CREATE TABLE "ProductRelease" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "productId" UUID NOT NULL REFERENCES "Product" (id),
    "submissionId" UUID NOT NULL UNIQUE REFERENCES "Submission" (id),
    "workflowRevisionId" UUID NOT NULL REFERENCES "WorkflowRevision" (id),
    version VARCHAR(100) NOT NULL,
    status "ProductReleaseStatus" NOT NULL DEFAULT 'PENDING_BUILD',
    changelog TEXT NOT NULL,
    "minimumLauncherVersion" VARCHAR(100) NOT NULL,
    "minimumRuntimeVersion" VARCHAR(100) NOT NULL,
    "workflowSchemaVersion" VARCHAR(100) NOT NULL,
    "packageObjectKey" TEXT UNIQUE,
    "packageSizeBytes" BIGINT CHECK ("packageSizeBytes" IS NULL OR "packageSizeBytes" >= 0),
    "packageHash" TEXT,
    "payloadHash" TEXT,

    -- Mock JSON: public package manifest; packageHash stays outside the package.
    -- {
    --   "formatVersion": "1.0",
    --   "productId": "uuid",
    --   "releaseId": "uuid",
    --   "version": "1.0.0",
    --   "entryWorkflowId": "main",
    --   "payload": {"path":"workflow.enc","hash":"sha256:..."}
    -- }
    manifest JSONB,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "algorithm": "AES-256-GCM",
    --   "keyId": "release-key-uuid",
    --   "nonceEncoding": "base64"
    -- }
    -- Không lưu plaintext encryption key.
    "encryptionMetadata" JSONB,

    -- Mock JSON:
    -- {"schemaVersion":"1.0","capabilities":["android.device.control"]}
    "requiredCapabilities" JSONB NOT NULL,

    -- Mock JSON Schema copied from approved revision for Launcher form generation.
    "inputSchema" JSONB NOT NULL,

    "publishedAt" TIMESTAMPTZ,
    "disabledAt" TIMESTAMPTZ,
    "disabledById" UUID REFERENCES "User" (id),
    "disabledReason" TEXT,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,

    CONSTRAINT "ProductRelease_product_version_key"
        UNIQUE ("productId", version),
    CONSTRAINT "ProductRelease_json_objects_check" CHECK (
        (manifest IS NULL OR jsonb_typeof(manifest) = 'object')
        AND ("encryptionMetadata" IS NULL OR jsonb_typeof("encryptionMetadata") = 'object')
        AND jsonb_typeof("requiredCapabilities") = 'object'
        AND jsonb_typeof("inputSchema") = 'object'
    ),
    CONSTRAINT "ProductRelease_published_artifact_check" CHECK (
        status <> 'PUBLISHED'
        OR (
            "packageObjectKey" IS NOT NULL
            AND "packageHash" IS NOT NULL
            AND "payloadHash" IS NOT NULL
            AND manifest IS NOT NULL
            AND "encryptionMetadata" IS NOT NULL
            AND "publishedAt" IS NOT NULL
        )
    ),
    CONSTRAINT "ProductRelease_disabled_fields_check" CHECK (
        status <> 'DISABLED'
        OR ("disabledAt" IS NOT NULL AND "disabledById" IS NOT NULL AND "disabledReason" IS NOT NULL)
    )
);

ALTER TABLE "Product"
ADD CONSTRAINT "Product_currentReleaseId_fkey"
FOREIGN KEY ("currentReleaseId") REFERENCES "ProductRelease" (id);
```

Build publication chỉ chuyển ProductRelease sang `PUBLISHED`; nó không đổi Product status hoặc
`currentReleaseId`. Transaction chọn current Release phải xác nhận Release thuộc đúng Product, đang
`PUBLISHED`, package hoàn chỉnh và chưa disable; FK một mình không enforce được các điều kiện đó.
Selection chỉ đổi `currentReleaseId` và Product audit fields, không đọc Offer, không đổi status và
không consume sliding-window slot.

Publisher lifecycle `ACTIVATE` lock Product, current ProductRelease, rồi ProductOffer; recheck
Release/Offer readiness, ghi initial Offer timestamp và chuyển `DRAFT -> ACTIVE`. Publisher
`ARCHIVED -> ACTIVE` và Admin `SUSPENDED -> ACTIVE` dùng cùng readiness checks nhưng không ghi Offer
timestamp. Admin approval chỉ cho phép build; Release selection chuẩn bị pointer và lifecycle action
mới quyết định Marketplace visibility. Nếu build/publish thất bại, Product và snapshot đang public
không đổi.

## 7. Quan hệ chính

```text
PublisherAccount
└── Product ── Project
    ├── ProductMedia[] ── Static
    ├── Submission[] ── WorkflowRevision
    │   ├── SubmissionScan[]
    │   │   └── SubmissionScanFinding[]
    │   └── SubmissionReview[]
    └── ProductRelease[]
```
