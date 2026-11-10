# 02 — Node Catalog, Project, Workflow Revision và Asset

## 1. Trách nhiệm nhóm

Nhóm này lưu phần cloud của Studio:

- Backend công bố Node Catalog hiện hành do hệ thống sở hữu; mỗi `type` có một definition mutable trong MVP.
- Studio tạo/sync mutable Project Draft.
- Asset binary nằm trong Object Storage; PostgreSQL lưu metadata và ownership.
- Khi publisher submit trên Web, Backend chốt đúng draft version thành immutable Workflow Revision.
- Submission review revision đó, không review draft đang tiếp tục thay đổi.

## 2. Capability vocabulary

### Trách nhiệm

`Capability` là vocabulary ổn định dùng chung cho Node Catalog, scan, release manifest và Runtime authorization.

```sql
CREATE TABLE "Capability" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    "isSensitive" BOOLEAN NOT NULL DEFAULT FALSE,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ
);
```

Capability đã có trong Node Catalog MVP:

```text
android.device.control
android.screen.capture
```

Capability dự kiến cho các phase sau:

```text
android.ui.observe
network.http
connector.google_sheets.write
```

## 3. Node Catalog

### Trách nhiệm

Feature contract và trạng thái triển khai: [`../studio_project_workflow/node_catalog.md`](../studio_project_workflow/node_catalog.md).

`NodeDefinition.type` là identity ổn định và unique. MVP chỉ lưu definition hiện hành; chưa có
`NodeDefinitionVersion`, node-level version, `$schema`, hoặc schema version lồng trong các JSON field.
System catalog publisher có quyền cập nhật definition và dùng `contentHash` để nhận biết thay đổi.

```sql
CREATE TYPE "NodeDefinitionStatus" AS ENUM (
    'ACTIVE',
    'DISABLED'
);

CREATE TABLE "NodeDefinition" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    category VARCHAR(100) NOT NULL,
    status "NodeDefinitionStatus" NOT NULL DEFAULT 'ACTIVE',
    "runtimeHandler" VARCHAR(255) NOT NULL,
    "minimumRuntimeVersion" VARCHAR(100) NOT NULL,
    "defaultTimeoutMs" INTEGER NOT NULL CHECK ("defaultTimeoutMs" > 0),
    "configSchema" JSONB NOT NULL,
    "inputSchema" JSONB NOT NULL,
    "outputSchema" JSONB NOT NULL,
    "outputPorts" JSONB NOT NULL,
    "editorHints" JSONB NOT NULL,
    "supportedErrorPolicies" JSONB NOT NULL,
    "contentHash" VARCHAR(64) NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,

    CONSTRAINT "NodeDefinition_contentHash_check"
        CHECK ("contentHash" ~ '^[0-9a-f]{64}$'),
    CONSTRAINT "NodeDefinition_json_objects_check" CHECK (
        jsonb_typeof("configSchema") = 'object'
        AND jsonb_typeof("inputSchema") = 'object'
        AND jsonb_typeof("outputSchema") = 'object'
        AND jsonb_typeof("outputPorts") = 'object'
        AND jsonb_typeof("editorHints") = 'object'
        AND jsonb_typeof("supportedErrorPolicies") = 'object'
    )
);

CREATE INDEX "NodeDefinition_status_type_idx"
ON "NodeDefinition" (status, type);

CREATE TABLE "NodeDefinitionCapability" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "nodeDefinitionId" UUID NOT NULL REFERENCES "NodeDefinition" (id),
    "capabilityId" UUID NOT NULL REFERENCES "Capability" (id),
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,
    CONSTRAINT "NodeDefinitionCapability_node_capability_key"
        UNIQUE ("nodeDefinitionId", "capabilityId")
);

CREATE INDEX "NodeDefinitionCapability_capabilityId_idx"
ON "NodeDefinitionCapability" ("capabilityId");
```

Publication validate toàn bộ source trước transaction, upsert definition theo `type`, đồng bộ capability,
và đổi các definition bị xóa khỏi source thành `DISABLED`. Studio chỉ nhận node `ACTIVE` tương thích với
`runtimeVersion`; `contentHash` toàn catalog hỗ trợ ETag/cache. Immutable node version và exact historical
resolution được hoãn đến khi MVP chứng minh cần thiết.

## 4. Cloud Project và mutable Draft

### Trách nhiệm

Project phase hiện tại: [`../studio_project_workflow/project_module.md`](../studio_project_workflow/project_module.md).
Project Draft contract: [`../studio_project_workflow/project_draft.md`](../studio_project_workflow/project_draft.md).

`Project` là container cloud thuộc đúng một `PublisherAccount`. Phase Project hiện tại chỉ triển khai
metadata, ownership và archive lifecycle; `ProjectDraft` là phase kế tiếp và giữ snapshot mutable mới
nhất với optimistic version để chống ghi đè giữa nhiều Studio session.

Quy tắc Project MVP: tên được trim, dài `1..255`, unique không phân biệt hoa thường theo
`PublisherAccount`; description nullable, được trim,
dài tối đa `4.000` ký tự và kết quả rỗng được lưu thành `NULL`; `ARCHIVED` vẫn đọc được nhưng không
được sửa; không có public delete, unarchive hoặc Redis lock cho CRUD thông thường.
Các giới hạn text và normalize được enforce bởi strict Zod API contract; phase này không lặp lại
chúng bằng database check constraint.

Authorization tách ownership khỏi capability: `PublisherAccount` không bị xóa khi publisher bị block
và tiếp tục làm ownership anchor. Khi block, Publisher `UserRole` bị soft-delete nhưng Customer role
được giữ lại; Customer có permission `OWN` để list/read project thuộc PublisherAccount của chính user.
Permission create không có scope và chỉ gán cho Publisher role; ownership luôn được suy ra từ
authenticated user và PublisherAccount `ACTIVE` của user đó. Publisher role `OWN` cho phép
update/archive, còn Admin permission có scope `NULL` để list/read/update/archive không giới hạn
owner. Scope `NULL` thắng `OWN` khi một user có nhiều role. Repository nhận typed access predicate;
chỉ `OWN` mới thêm `publisherAccountId` vào query. Mutation `OWN` yêu cầu PublisherAccount `ACTIVE`;
Admin không cần resolve hoặc kiểm tra trạng thái PublisherAccount của owner nhưng vẫn tuân theo
lifecycle của Project.

```sql
CREATE TYPE "ProjectStatus" AS ENUM (
    'ACTIVE',
    'ARCHIVED'
);

CREATE TABLE "Project" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "publisherAccountId" UUID NOT NULL REFERENCES "PublisherAccount" (id) ON DELETE NO ACTION,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    status "ProjectStatus" NOT NULL DEFAULT 'ACTIVE',
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,
    "createdById" UUID NOT NULL REFERENCES "User" (id) ON DELETE NO ACTION,
    "updatedById" UUID REFERENCES "User" (id) ON DELETE NO ACTION,
    "deletedById" UUID REFERENCES "User" (id) ON DELETE NO ACTION
);

CREATE INDEX "Project_publisherAccountId_status_idx"
ON "Project" ("publisherAccountId", status)
WHERE "deletedAt" IS NULL;

CREATE UNIQUE INDEX "Project_publisherAccountId_name_ci_key"
ON "Project" ("publisherAccountId", lower(btrim(name)));

CREATE TABLE "ProjectDraft" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "projectId" UUID NOT NULL UNIQUE REFERENCES "Project" (id) ON DELETE NO ACTION,
    version INTEGER NOT NULL DEFAULT 1 CHECK (version > 0),

    -- Mock JSON: editable Studio graph, including editor-only position metadata.
    -- {
    --   "schemaVersion": "1.0",
    --   "entryNodeId": "start_01",
    --   "nodes": [
    --     {
    --       "id": "tap_01",
    --       "type": "android.tap",
    --       "config": {},
    --       "bindings": {},
    --       "editor": {"x": 420, "y": 180}
    --     }
    --   ],
    --   "edges": []
    -- }
    "editorGraph" JSONB NOT NULL,

    -- Mock Backend-owned input schema envelope:
    -- {
    --   "type": "object",
    --   "required": ["accountName"],
    --   "properties": {"accountName": {"type": "string"}}
    -- }
    "inputSchema" JSONB NOT NULL,

    -- Mock JSON: compiler output cached for submission preparation; Backend revalidates on finalize.
    -- {
    --   "irVersion": "1.0",
    --   "entryNodeId": "start_01",
    --   "nodes": [],
    --   "requiredCapabilities": ["android.device.control"]
    -- }
    "runtimeIr" JSONB,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "status": "VALID_WITH_WARNINGS",
    --   "errors": [],
    --   "warnings": [{"code": "UNTESTED_BRANCH", "nodeId": "if_01"}]
    -- }
    "validationSummary" JSONB NOT NULL DEFAULT '{}'::JSONB,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "runtimeVersion": "0.1.0",
    --   "lastTestedAt": "2026-10-04T10:00:00Z",
    --   "successfulRuns": 2,
    --   "failedRuns": 0
    -- }
    "testSummary" JSONB NOT NULL DEFAULT '{}'::JSONB,

    "contentHash" VARCHAR(64) NOT NULL,
    "savedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "savedById" UUID NOT NULL REFERENCES "User" (id) ON DELETE NO ACTION,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,

    CONSTRAINT "ProjectDraft_contentHash_check"
        CHECK ("contentHash" ~ '^[0-9a-f]{64}$'),

    CONSTRAINT "ProjectDraft_json_objects_check" CHECK (
        jsonb_typeof("editorGraph") = 'object'
        AND jsonb_typeof("inputSchema") = 'object'
        AND ("runtimeIr" IS NULL OR jsonb_typeof("runtimeIr") = 'object')
        AND jsonb_typeof("validationSummary") = 'object'
        AND jsonb_typeof("testSummary") = 'object'
    )
);
```

Draft được tạo lazy: `expectedVersion = 0` chỉ dùng cho lần save đầu và tạo version `1`; các lần
save sau gửi version hiện tại. Conditional update chỉ tăng version khi content hash thay đổi. Retry
cùng content hash là idempotent; stale version với content khác trả conflict cùng current version.
Backend canonicalize năm field content và tự tính SHA-256; Studio không gửi hash/audit field. Tổng
canonical UTF-8 content tối đa 1 MiB. API `GET` trả content và metadata đồng bộ nhưng không expose
`savedById`; API `PUT` chỉ trả `id`, `version`, `contentHash`, `savedAt` để Studio tiếp tục optimistic
save mà không echo toàn bộ payload. Conflict trả `details.currentVersion`. Project và ProjectDraft
dùng chung một access policy; read cho phép `NULL` hoặc `OWN`, còn save draft bắt buộc `OWN` và
PublisherAccount `ACTIVE`. Web submission phải gửi exact draft version đã chọn để submit.

Studio chỉ tạo Project cloud khi user chọn Save to Cloud lần đầu, sau đó lưu ngay draft hiện tại;
Backend không tạo draft rỗng cùng Project. Cloud là source of truth cho state đã sync và thiết bị mới
phải download Project + draft. `GET draft` luôn trả thêm `projectStatus` để Studio chuyển sang
read-only khi archive, nhưng Backend vẫn kiểm tra lifecycle lại trên từng lần save.

## 5. Asset

### Trách nhiệm

`Asset` lưu metadata/ownership của object; binary nằm ở Object Storage. `ProjectAsset` gắn asset mutable với project. Revision sẽ snapshot asset bằng mapping riêng.

```sql
CREATE TYPE "AssetType" AS ENUM (
    'IMAGE_TEMPLATE',
    'OCR_RESOURCE',
    'ICON',
    'SCREENSHOT',
    'VIDEO',
    'OTHER'
);

CREATE TYPE "AssetStatus" AS ENUM (
    'UPLOADING',
    'READY',
    'QUARANTINED',
    'DELETED'
);

CREATE TABLE "Asset" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "ownerUserId" UUID NOT NULL REFERENCES "User" (id),
    type "AssetType" NOT NULL,
    status "AssetStatus" NOT NULL DEFAULT 'UPLOADING',
    "objectKey" TEXT NOT NULL UNIQUE,
    "originalFileName" TEXT NOT NULL,
    "contentType" VARCHAR(255) NOT NULL,
    "sizeBytes" BIGINT NOT NULL CHECK ("sizeBytes" >= 0),
    "sha256" TEXT,

    -- Mock JSON:
    -- {
    --   "schemaVersion": "1.0",
    --   "width": 1080,
    --   "height": 1920,
    --   "durationMs": null,
    --   "ocrLanguages": []
    -- }
    "mediaMetadata" JSONB NOT NULL DEFAULT '{}'::JSONB,

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,

    CONSTRAINT "Asset_mediaMetadata_object_check"
        CHECK (jsonb_typeof("mediaMetadata") = 'object')
);

CREATE TABLE "ProjectAsset" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "projectId" UUID NOT NULL REFERENCES "Project" (id),
    "assetId" UUID NOT NULL REFERENCES "Asset" (id),
    alias VARCHAR(255) NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "deletedAt" TIMESTAMPTZ
);

CREATE UNIQUE INDEX "ProjectAsset_project_alias_idx"
ON "ProjectAsset" ("projectId", lower(alias))
WHERE "deletedAt" IS NULL;

CREATE UNIQUE INDEX "ProjectAsset_project_asset_idx"
ON "ProjectAsset" ("projectId", "assetId")
WHERE "deletedAt" IS NULL;
```

## 6. Immutable Workflow Revision

### Trách nhiệm

Feature contract draft để review:
[`../studio_project_workflow/workflow_revision.md`](../studio_project_workflow/workflow_revision.md).

`WorkflowRevision` là snapshot Backend tạo từ đúng ProjectDraft version lúc submit trên Web. Record
này không update content và không soft delete. Vì Node Catalog MVP chỉ giữ definition hiện hành,
revision phải freeze đầy đủ contract của từng node type đã dùng; chỉ lưu `type + contentHash` không
đủ để resolve definition lịch sử.

```sql
CREATE TABLE "WorkflowRevision" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "projectId" UUID NOT NULL REFERENCES "Project" (id),
    "sourceDraftVersion" INTEGER NOT NULL CHECK ("sourceDraftVersion" > 0),
    "sourceDraftContentHash" VARCHAR(64) NOT NULL,

    -- Mock JSON: frozen Editor Graph, giữ metadata để admin/publisher inspect.
    -- {"schemaVersion":"1.0","entryNodeId":"start_01","nodes":[],"edges":[]}
    "editorGraph" JSONB NOT NULL,

    -- Mock JSON: canonical executable representation.
    -- {
    --   "irVersion": "1.0",
    --   "entryNodeId": "start_01",
    --   "nodes": [
    --     {
    --       "id": "start_01",
    --       "handler": {"type": "flow.start"},
    --       "bindings": {},
    --       "transitions": {"success": "end_01"}
    --     }
    --   ]
    -- }
    "runtimeIr" JSONB NOT NULL,

    -- Mock JSON Schema: same meaning as ProjectDraft.inputSchema.
    "inputSchema" JSONB NOT NULL,

    -- Full immutable snapshot của distinct Node Definitions, sort theo type.
    -- Mock JSON:
    -- {
    --   "schemaVersion": 1,
    --   "definitions": [
    --     {
    --       "type": "android.tap",
    --       "runtimeHandler": "android.tap",
    --       "minimumRuntimeVersion": "0.1.0",
    --       "configSchema": {},
    --       "inputSchema": {},
    --       "outputSchema": {},
    --       "outputPorts": {},
    --       "editorHints": {},
    --       "supportedErrorPolicies": {},
    --       "capabilityCodes": ["android.device.control"],
    --       "contentHash": "64-char-lowercase-sha256"
    --     }
    --   ]
    -- }
    "nodeDefinitionManifest" JSONB NOT NULL,

    -- Mock JSON:
    -- {
    --   "schemaVersion": 1,
    --   "capabilities": ["android.device.control", "android.ui.observe"]
    -- }
    "capabilityManifest" JSONB NOT NULL,

    -- Studio-provided advisory snapshot; cloud validation is stored by Submission pipeline.
    -- Mock JSON:
    -- {"schemaVersion":"1.0","status":"VALID","errors":[],"warnings":[]}
    "validationSummary" JSONB NOT NULL,

    -- Mock JSON:
    -- {"schemaVersion":"1.0","runtimeVersion":"0.1.0","successfulRuns":2}
    "testSummary" JSONB NOT NULL DEFAULT '{}'::JSONB,

    "contentHash" VARCHAR(64) NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "createdById" UUID NOT NULL REFERENCES "User" (id),

    CONSTRAINT "WorkflowRevision_project_draftVersion_key"
        UNIQUE ("projectId", "sourceDraftVersion"),
    CONSTRAINT "WorkflowRevision_sourceDraftContentHash_check"
        CHECK ("sourceDraftContentHash" ~ '^[0-9a-f]{64}$'),
    CONSTRAINT "WorkflowRevision_contentHash_check"
        CHECK ("contentHash" ~ '^[0-9a-f]{64}$'),
    CONSTRAINT "WorkflowRevision_json_objects_check" CHECK (
        jsonb_typeof("editorGraph") = 'object'
        AND jsonb_typeof("runtimeIr") = 'object'
        AND jsonb_typeof("inputSchema") = 'object'
        AND jsonb_typeof("nodeDefinitionManifest") = 'object'
        AND jsonb_typeof("capabilityManifest") = 'object'
        AND jsonb_typeof("validationSummary") = 'object'
        AND jsonb_typeof("testSummary") = 'object'
    )
);

CREATE TABLE "WorkflowRevisionAsset" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "workflowRevisionId" UUID NOT NULL REFERENCES "WorkflowRevision" (id),
    "assetId" UUID NOT NULL REFERENCES "Asset" (id),
    alias VARCHAR(255) NOT NULL,
    "assetHash" VARCHAR(64) NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT "WorkflowRevisionAsset_revision_alias_key"
        UNIQUE ("workflowRevisionId", alias),
    CONSTRAINT "WorkflowRevisionAsset_revision_asset_key"
        UNIQUE ("workflowRevisionId", "assetId"),
    CONSTRAINT "WorkflowRevisionAsset_assetHash_check"
        CHECK ("assetHash" ~ '^[0-9a-f]{64}$')
);
```

`sourceDraftContentHash` giữ provenance của draft đã chọn. `contentHash` là SHA-256 của canonical
revision content gồm graph, IR, input schema, full node manifest, capability manifest,
validation/test summary và asset alias/hash; identity/audit field không tham gia hash.

`WorkflowRevisionAsset.assetHash` đóng băng expected content. Asset object không được overwrite;
thay đổi asset tạo object/Asset mới. Quan hệ này chỉ được migrate sau khi Asset module tồn tại. Trong
phase asset-free, revision hash dùng `assets: []` và submission có asset reference phải bị reject.

## 7. Quan hệ chính

```text
Capability
└── NodeDefinitionCapability
    └── NodeDefinition

PublisherAccount
└── Project
    ├── ProjectDraft                 # mutable, one current row
    ├── ProjectAsset[] → Asset
    └── WorkflowRevision[]           # immutable
        └── WorkflowRevisionAsset[] → Asset
```

## 8. Không lưu trong Backend DB

- Studio viewport history, undo stack hoặc local crash-recovery files.
- Test credentials và raw secrets.
- Asset binary/blob.
- Runtime development execution detail; Runtime Agent giữ local execution state.
