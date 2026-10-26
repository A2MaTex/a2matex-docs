## 3. Mô hình sản phẩm

Cần phân biệt ba khái niệm chính:

| Thành phần       | Ý nghĩa                                                                    |
| ---------------- | -------------------------------------------------------------------------- |
| `Product`        | Editable aggregate Publisher sở hữu: identity, presentation data và media |
| `Submission`     | Immutable Product + WorkflowRevision + release-request snapshot để review |
| `ProductRelease` | Phiên bản kỹ thuật immutable đã được duyệt, build và publish               |

Ví dụ:

```text
Product: Auto mở rương game A
├── Draft fields: tên, mô tả, hướng dẫn, Static-backed image media
├── Release 1.0.0
├── Release 1.1.0
└── Release 1.2.0 — current release
```

### 3.1. Product aggregate

Product là working copy Publisher có thể lưu trước khi submit:

- Tên, slug, mô tả ngắn và mô tả chi tiết.
- Hướng dẫn sử dụng.
- Ordered image media (`ICON`, `COVER`, `GALLERY`) tham chiếu ảnh đã upload qua Static/R2.
- Publisher.
- Trạng thái lifecycle.
- Current release.

MVP không có ProductListing, Category, Tag hoặc ProductTag riêng. Giá/chính sách bán thuộc Product
Offer. Mọi thay đổi presentation hoặc media muốn xuất hiện trên Marketplace đều cần Submission mới;
ProductRelease hiện tại vẫn public trong lúc Publisher sửa draft hoặc chờ review.

### 3.2. ProductRelease

Chứa thông tin kỹ thuật immutable:

- Version và changelog.
- Package object key và package hash.
- Workflow schema version.
- Input/output schema.
- Required capabilities.
- Minimum Launcher version.
- Minimum Runtime version.
- Payload hash và encryption metadata.
- Release notes và ngày phát hành.

ProductRelease đã publish không được ghi đè. Muốn sửa workflow phải tạo submission và release mới.

## 4. Submission từ Publisher Portal

Studio chỉ save/sync Project draft. Publisher tạo/chỉnh Product và bấm `Submit for Review` trên Web
Publisher Portal. Backend lock Product hiện tại và exact Project Draft version để tạo snapshot.

### 4.1. Product metadata

- Product ID.
- Product name, slug, descriptions và usage instructions.
- Ordered image media metadata + `staticId`; không snapshot private URL/R2 object key.
- Terms hoặc cảnh báo sử dụng.

### 4.2. Release snapshot

- Version và changelog.
- Project snapshot.
- Workflow graph hoặc Runtime IR.
- Input/output schema.
- Required capabilities.
- Minimum Launcher/Runtime version.
- Assets.
- Known limitations.

### 4.3. Review metadata

- Tài khoản hoặc môi trường test nếu cần.
- LDPlayer và Android version đã test.
- Test instruction.
- Mục đích workflow.
- Danh sách hành động và dữ liệu mà workflow sử dụng.

### 4.4. Tính bất biến

Submission đã gửi là immutable. Khi admin yêu cầu chỉnh sửa:

```text
Submission 01: CHANGES_REQUESTED
→ Publisher sửa Product trên Web và/hoặc project trong Studio
→ Publisher tạo Submission 02 trên Web
→ Submission 02 tham chiếu Submission 01
→ Admin review snapshot mới
```

Không cập nhật snapshot của submission cũ.

## 5. Quy trình kiểm duyệt

```mermaid
flowchart TD
    Submit["Publisher tạo submission"] --> Scan["Kiểm tra tự động"]
    Scan --> Review["Admin review"]
    Review --> Change["Yêu cầu chỉnh sửa"]
    Review --> Reject["Từ chối"]
    Review --> Approve["Phê duyệt"]
    Change --> New["Tạo submission mới"]
    New --> Scan
    Approve --> Release["Build ProductRelease"]
```

### 5.1. Kiểm tra tự động

Backend hoặc review worker kiểm tra:

- Graph và schema hợp lệ.
- Có entry node và đường đi hợp lệ.
- Node type/version đều được hỗ trợ.
- Không có cycle hoặc loop không được kiểm soát.
- Input/output schema hợp lệ.
- Capability nằm trong allowlist.
- Code node tuân thủ giới hạn của hệ thống.
- Không có raw shell/ADB command do publisher tự chèn.
- Asset đầy đủ, đúng loại và không vượt kích thước.
- Package không chứa file ngoài format cho phép.
- Launcher/Runtime version tương thích.
- Test compile thành Runtime IR thành công.

Kết quả scan được lưu theo từng rule để admin biết lỗi nào chặn publish và lỗi nào chỉ là cảnh báo.

### 5.2. Admin review thủ công

Admin kiểm tra:

- Product presentation snapshot có phản ánh đúng hành vi workflow không.
- Workflow có hành vi nguy hiểm hoặc gây hiểu nhầm không.
- Có thu thập hoặc truyền dữ liệu nhạy cảm không.
- Capability yêu cầu có hợp lý không.
- Screenshot và video có đúng sản phẩm không.
- Workflow chạy được theo test instruction không.
- Sản phẩm có vi phạm điều khoản nền tảng không.

### 5.3. Trạng thái submission

| Trạng thái          | Ý nghĩa                                  |
| ------------------- | ---------------------------------------- |
| `PENDING`           | Đang chờ hoặc đang được review           |
| `CHANGES_REQUESTED` | Cần tạo submission mới sau khi chỉnh sửa |
| `APPROVED`          | Snapshot đã được duyệt                   |
| `REJECTED`          | Bị từ chối                               |

MVP chưa cần thêm nhiều trạng thái nội bộ. Scan status có thể lưu riêng với submission status.

## 6. Build và publish ProductRelease

Khi admin approve:

1. Backend xác nhận submission snapshot đã bị khóa.
2. Chạy lại validation trên đúng snapshot được duyệt.
3. Compile graph thành Runtime IR chuẩn.
4. Tạo content encryption key cho release.
5. Mã hóa workflow payload.
6. Đóng gói artifact dạng zip .a2m.
7. Tính package hash và lưu ngoài package.
8. Upload package lên Object Storage.
9. Tạo immutable `ProductRelease`.
10. Gán release làm current release khi publisher/admin chọn publish.

Cấu trúc package:

```text
product.a2m
├── manifest.json
├── workflow.enc
├── input-schema.json
└── assets/
```

Ví dụ `manifest.json`:

```json
{
  "productId": "product_01",
  "releaseId": "release_01",
  "version": "1.0.0",
  "workflowSchemaVersion": "1.0",
  "minimumLauncherVersion": "1.0.0",
  "minimumRuntimeVersion": "1.0.0",
  "entryWorkflowId": "main",
  "payloadHash": "sha256:...",
  "encryptionAlgorithm": "AES-256-GCM"
}
```

`packageHash` của toàn bộ `.a2m` được lưu trong `ProductRelease` và download metadata, không đặt vào chính package để tránh tự tham chiếu.

Hash dùng để:

- Phát hiện package bị thay đổi hoặc download thiếu.
- Xác định đúng release.
- Tránh chạy nhầm artifact.

Encryption chỉ làm tăng chi phí đọc workflow, không thể chống tuyệt đối reverse engineering trên máy customer.
Release content key được Backend key service bảo vệ và chỉ được wrap vào
Runtime-bound key envelope sau khi authorization thành công; Launcher không nhận
plaintext key. Xem
[Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).
