# 6. Artifact sau khi được Approve

Không build thành:

- APK.
- APKG.
- JavaScript executable gửi cho customer.
- Flutter bundle.
- Native executable.

Sản phẩm được build thành một `ProductRelease` bất biến, chỉ lưu trên server.

```text
ProductRelease
├── Release manifest
├── Execution IR
├── Script artifacts
├── Asset references
├── Capability declaration
├── Runtime compatibility
└── Artifact hash
```

Có thể biểu diễn:

```json
{
  "releaseId": "release_01J...",
  "productId": "product_01J...",
  "version": "1.0.0",
  "compilerVersion": "1.0.0",
  "irVersion": 1,
  "artifactHash": "sha256:...",
  "entryStepId": "step_start",
  "executionIr": {},
  "scriptArtifacts": [],
  "requiredCapabilities": ["android.accessibility"],
  "targetPackages": ["com.example.target"]
}
```

## Vì sao build trước khi admin approve?

Admin phải review và test đúng artifact sẽ được publish.

Luồng đúng:

```text
Submission
→ Server build candidate
→ Admin review candidate
→ Approve
→ Promote đúng candidate thành ProductRelease
```

Không nên approve source A rồi build lại thành artifact B mà admin chưa từng xem.

Build phải deterministic:

```text
Cùng source
+ cùng compiler version
= cùng artifact hash
```

# 7. Lưu ProductRelease ở đâu?

MVP có thể lưu:

| Dữ liệu                  | Nơi lưu          |
| ------------------------ | ---------------- |
| Release metadata         | PostgreSQL       |
| Execution IR             | PostgreSQL JSONB |
| Script source để audit   | Object storage   |
| Compiled script artifact | Object storage   |
| Icon và screenshot       | Object storage   |
| Artifact hash            | PostgreSQL       |

Execution IR không được dùng trong API catalog thông thường.

Trang marketplace chỉ đọc:

- Product name.
- Description.
- Price.
- Icon.
- Rating nếu có.
- Published version.
- Publisher.

Do đó kích thước IR không ảnh hưởng trực tiếp tới tốc độ hiển thị marketplace.

# 8. Product và ProductRelease khác nhau thế nào?

## Product

Là danh tính lâu dài của sản phẩm:

```text
Product
├── Name
├── Publisher
├── Description
├── Category
├── Pricing plan
├── Current published release
└── Marketplace status
```

## ProductRelease

Là một phiên bản đã được build và duyệt:

```text
ProductRelease 1.0.0
ProductRelease 1.1.0
ProductRelease 2.0.0
```

Product trỏ đến release đang được sử dụng:

```text
Product.currentReleaseId → ProductRelease.id
```

Khi publisher phát hành phiên bản mới:

- Tạo submission mới.
- Build release mới.
- Admin review.
- Nếu approve, cập nhật `currentReleaseId`.
- Release cũ vẫn được giữ để audit hoặc rollback.

# 9. Product Status

MVP chỉ cần:

| Status      | Ý nghĩa                                            |
| ----------- | -------------------------------------------------- |
| `ACTIVE`    | Hiển thị và có thể sử dụng                         |
| `SUSPENDED` | Ẩn khỏi marketplace và không cho tạo execution mới |

Không cần xóa ProductRelease đã publish.

Nếu phát hiện release có lỗi nghiêm trọng:

- Suspend product.
- Dừng execution mới.
- Có thể đóng execution đang chạy.
- Chuyển `currentReleaseId` về release an toàn trước đó.
