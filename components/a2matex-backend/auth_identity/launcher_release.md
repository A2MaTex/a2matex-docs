# A2MaTex Backend — Launcher Release Module

## 1. Mục tiêu

Launcher Release Module quản lý package Launcher hiện tại mà người dùng có thể tải xuống.

- Không gắn một Launcher installation hoặc một Launcher version với user.
- Không giới hạn số máy hoặc số lần tải xuống trong giai đoạn hiện tại.
- Không lưu device fingerprint, installation status hoặc Launcher client key.
- Package vật lý được lưu trên object storage thông qua `Static`; module này chỉ lưu version và package nào đang active.

Thiết kế này thay thế phạm vi `Installation` đã mô tả trước đây. Backend chưa cần `Installation` table hoặc API đăng ký installation.

## 2. Quan hệ dữ liệu

```text
LauncherRelease
└── staticId → Static → object storage (Cloudflare R2 hoặc S3-compatible storage)
```

`Static` chịu trách nhiệm lưu metadata kỹ thuật của file:

- object key;
- original file name;
- content type;
- file size.

`LauncherRelease` chịu trách nhiệm lưu ý nghĩa nghiệp vụ của package:

- Launcher version;
- release notes;
- package được liên kết;
- release nào đang active;
- thời điểm release được publish.

## 3. Database schema dự kiến

```sql
CREATE TABLE "LauncherRelease" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version VARCHAR(100) NOT NULL,
    "staticId" UUID NOT NULL REFERENCES "Static" (id),
    "releaseNotes" TEXT,
    "isActive" BOOLEAN NOT NULL DEFAULT FALSE,
    "publishedAt" TIMESTAMPTZ,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ
);
```

Không cần enum cho module này. Backend chịu trách nhiệm kiểm tra business constraints.

Quy tắc:

- Tại một thời điểm chỉ có một `LauncherRelease` chưa bị xóa được active.
- Release mới được tạo ở trạng thái inactive.
- Activate một release phải deactivate release hiện tại trước, sau đó activate release được chọn trong cùng transaction.
- Khi activate, đặt `publishedAt` nếu release chưa từng được publish.
- Release cũ được giữ lại để admin có thể rollback bằng cách activate lại.
- `staticId` phải trỏ tới một `Static` chưa bị xóa.
- Không xóa file `Static` khi chỉ deactivate một release.

## 4. API contract

Các response thành công vẫn được global interceptor wrap theo format chung của backend. Các ví dụ dưới đây mô tả inner payload.

### 4.1. Lấy Launcher release hiện tại

```http
GET /launcher-releases/current
```

Public API, không yêu cầu access token.

Response:

```json
{
  "id": "2b22e5c5-211c-4891-87eb-40f7b5815f0d",
  "version": "1.0.0",
  "staticId": "6de0bc89-9d83-43ea-b34f-d55a644fab0c",
  "releaseNotes": "Initial MVP release",
  "publishedAt": "2026-10-05T10:00:00.000Z"
}
```

Client tạm thời dùng `staticId` với API `GET /statics/files/{staticId}` hiện có để tải package. Không trả object key hoặc R2 URL.

Nếu chưa có active release, trả lỗi not found theo error convention của backend.

### 4.2. Admin tạo Launcher release

```http
POST /launcher-releases
Authorization: Bearer <access-token>
```

Request:

```json
{
  "version": "1.0.0",
  "staticId": "6de0bc89-9d83-43ea-b34f-d55a644fab0c",
  "releaseNotes": "Initial MVP release"
}
```

Package phải được upload trước bằng Static upload API hiện có.

Response chỉ trả ID đã tạo:

```json
{
  "id": "2b22e5c5-211c-4891-87eb-40f7b5815f0d"
}
```

Create không tự động activate release.

### 4.3. Admin lấy danh sách Launcher releases

```http
GET /launcher-releases?page=1&pageSize=20
Authorization: Bearer <access-token>
```

Sử dụng shared pagination parser của backend. Query không hợp lệ dùng giá trị mặc định theo convention hiện có.

Mỗi item gồm:

```json
{
  "id": "2b22e5c5-211c-4891-87eb-40f7b5815f0d",
  "version": "1.0.0",
  "staticId": "6de0bc89-9d83-43ea-b34f-d55a644fab0c",
  "originalFileName": "a2matex-launcher-1.0.0.zip",
  "size": 104857600,
  "isActive": true,
  "publishedAt": "2026-10-05T10:00:00.000Z",
  "createdAt": "2026-10-05T09:30:00.000Z"
}
```

### 4.4. Admin xem chi tiết Launcher release

```http
GET /launcher-releases/{launcherReleaseId}
Authorization: Bearer <access-token>
```

Response bao gồm toàn bộ release fields và Static metadata cần thiết cho màn hình chỉnh sửa. Không trả object-storage credentials hoặc internal object URL.

### 4.5. Admin cập nhật Launcher release

```http
PUT /launcher-releases/{launcherReleaseId}
Authorization: Bearer <access-token>
```

Request fields đều optional, nhưng request phải có ít nhất một field:

```json
{
  "version": "1.0.1",
  "staticId": "9b7b6ac1-bbbf-4cf4-8ee7-089b19082a98",
  "releaseNotes": "Replace the Launcher package"
}
```

Backend kiểm tra `staticId` mới tồn tại và chưa bị xóa trước khi cập nhật.

### 4.6. Admin activate Launcher release

```http
PUT /launcher-releases/{launcherReleaseId}/activate
Authorization: Bearer <access-token>
```

Không có request body.

Transaction:

```text
1. Kiểm tra release được chọn tồn tại và chưa bị xóa.
2. Kiểm tra Static package tương ứng tồn tại và chưa bị xóa.
3. Nếu release đã active thì trả lỗi already active.
4. Deactivate release đang active.
5. Activate release được chọn và đặt publishedAt nếu đang null.
6. Commit.
```

Response dùng standard success payload của backend.

## 5. Authorization

- API `GET /launcher-releases/current` là public.
- Create, list, detail, update và activate là protected routes.
- Permission được tạo từ controller decorators theo permission seed flow hiện có.
- API tự có khả năng quản lý mọi Launcher release; role nào được phép gọi do `RolePermission` quyết định.

## 6. Edge cases

| Trường hợp | Xử lý |
| --- | --- |
| Chưa có active release | Public current trả not found |
| `staticId` không tồn tại hoặc đã bị xóa | Create/update/activate bị từ chối |
| Activate release đang active | Trả conflict/already active |
| Activate release cũ | Cho phép; release active hiện tại bị deactivate |
| Hai request activate đồng thời | Serialize trong transaction để kết quả cuối chỉ có một active release |
| Storage object không tồn tại | Download trả storage/not-found error và ghi log theo convention hiện có |
| Update release active | Cho phép trong MVP; thay đổi có hiệu lực ngay trên current metadata/download |

## 7. Ngoài phạm vi hiện tại

- Installation registration và device ownership.
- Giới hạn số máy hoặc số lượt download.
- Machine fingerprint.
- Launcher auto-update protocol.
- Platform/CPU architecture variants.
- Minimum supported Launcher hoặc Runtime version.
- Mandatory upgrade policy.
- Release channels như stable, beta hoặc nightly.
- Presigned download URL.
- Download analytics.

## 8. Quyết định đã chốt

1. Xóa `Installation` khỏi phạm vi MVP hiện tại.
2. User tạm thời tải Launcher qua Static get-file API; download không tạo installation record.
3. Runtime được đóng gói cùng Launcher package.
4. File vật lý lưu qua `Static` và object-storage abstraction hiện có.
5. `LauncherRelease` lưu version và xác định package active hiện tại.
6. Giữ release cũ để có thể rollback bằng cách activate lại.
7. Không dùng enum cho module này.
