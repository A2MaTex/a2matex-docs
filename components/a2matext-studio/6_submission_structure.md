# 6. Submission gồm những gì?

Submission là immutable snapshot được tạo từ một cloud draft revision cụ thể.

Nó không phải APK hay executable.

## 6.1 Cấu trúc logic

```text
submission/
├── manifest.json
├── project.graph.json
├── workflow.ir.json
├── assets/
├── test-report.json
└── checksums.json
```

Trong API có thể upload dạng ZIP. Server giải nén, validate rồi lưu các thành phần riêng. Không cần giữ ZIP làm định dạng runtime lâu dài.

## 6.2 Manifest bắt buộc

```json
{
  "productId": "product_01J...",
  "projectId": "proj_01J...",
  "draftRevision": 12,
  "version": "1.0.0",
  "schemaVersion": 1,
  "compilerVersion": "1.0.0",
  "minimumRuntimeVersion": "1.0.0",
  "entryWorkflowId": "main",
  "capabilities": ["android.accessibility", "android.input-text"],
  "targetPackages": ["com.example.target"],
  "limits": {
    "maxDurationMs": 300000,
    "maxSteps": 500
  }
}
```

## 6.3 Marketplace metadata

- Product name.
- Short description.
- Full description.
- Category.
- Icon.
- Screenshots.
- Release notes.
- Support contact.
- Documentation URL.
- Version.
- Pricing proposal.
- Supported Android versions.
- Supported applications/package names.
- Language.

## 6.4 Execution declaration

- Required capabilities.
- Input parameters.
- Output schema.
- Target packages.
- Expected permissions.
- Maximum runtime.
- Retry/repeat limits.
- Supported Runtime Agent version.

## 6.5 Compliance declaration

- Loại dữ liệu được đọc.
- Loại dữ liệu được nhập.
- Có thu thập dữ liệu người dùng không.
- Có chụp màn hình không.
- Có đọc clipboard không.
- Có truy cập external endpoint không.
- Lý do sử dụng từng capability.
- Privacy policy nếu cần.

## 6.6 Test evidence

- Studio version.
- Compiler version.
- Device model dùng để test.
- Android version.
- Thời điểm test gần nhất.
- Test result.
- Node coverage cơ bản.
- Validation warnings còn tồn tại.

Server không được tin trực tiếp `workflow.ir.json` do Studio gửi. Studio gửi cả source graph và IR để hỗ trợ review, nhưng server phải:

1. Validate lại graph.
2. Dùng đúng compiler version để compile lại.
3. So sánh content hash.
4. Từ chối nếu IR phía Studio khác IR server tạo ra.
