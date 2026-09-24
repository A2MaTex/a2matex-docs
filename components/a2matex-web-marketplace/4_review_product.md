# 5. Quy trình Review

## 5.1. Automated Pre-check

Ngay khi nhận submission, backend tự động kiểm tra:

- Publisher sở hữu product và project.
- Draft revision tồn tại.
- Source hash hợp lệ.
- Graph schema hợp lệ.
- Có đúng một Start node.
- Node và edge reference hợp lệ.
- Không có unreachable node nghiêm trọng.
- Node version được hỗ trợ.
- JavaScript parse thành công.
- JavaScript không sử dụng API bị cấm.
- Input/output schema hợp lệ.
- Capability được khai báo đầy đủ.
- Target package hợp lệ.
- Asset tồn tại.
- Runtime limit nằm trong giới hạn.
- Version chưa tồn tại.

Nếu pre-check thất bại, submission không cần vào hàng chờ admin.

## 5.2. Server Build Candidate

Server không tin IR hoặc JavaScript artifact do Studio tạo.

Backend sử dụng trusted compiler để:

```text
Source graph
→ Validate
→ Normalize
→ Compile JavaScript
→ Resolve transitions
→ Generate Execution IR
→ Generate artifact hash
```

Kết quả là một candidate release artifact để admin review.

```text
Candidate Release Artifact
├── manifest.json
├── execution-ir.json
├── scripts/
│   └── compiled-script artifacts
└── checksums.json
```

Candidate này chưa được customer sử dụng.

## 5.3. Manual Review

Admin Console hiển thị:

### Product information

- Product name.
- Publisher.
- Version.
- Description.
- Category.
- Icon và screenshot.
- Release notes.

### Technical information

- Workflow graph dạng read-only.
- Danh sách node.
- JavaScript source.
- Execution targets.
- Android capabilities.
- Target Android packages.
- Inputs và outputs.
- Runtime limits.
- Background connectors.
- Compiler warnings.

### Risk information

- Có sử dụng Accessibility Service không.
- Có nhập dữ liệu vào app khác không.
- Có đọc text trên màn hình không.
- Có sử dụng coordinate fallback không.
- Có gửi dữ liệu sang external service không.
- Có script JavaScript không.
- Có node bị giới hạn capability không.

### Test information

- Publisher đã test trên thiết bị nào.
- Android version.
- Kết quả test gần nhất.
- Node nào chưa được chạy qua.
- Validation warnings.

## 5.4. Quyết định Review

Admin có ba action:

### Approve

- Candidate artifact được đánh dấu là release chính thức.
- Tạo immutable `ProductRelease`.
- Product được đưa lên marketplace hoặc cập nhật phiên bản mới.

### Request Changes

- Submission chuyển thành `CHANGES_REQUESTED`.
- Admin nhập review note.
- Publisher sửa cloud draft trong Studio.
- Publisher tạo submission mới.

Submission cũ không bị sửa hoặc ghi đè.

### Reject

- Submission chuyển thành `REJECTED`.
- Lưu lý do từ chối.
- Không tạo published release.
