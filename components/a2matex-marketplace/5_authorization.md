## 10. Quản lý quyền sử dụng

Trạng thái entitlement cho MVP:

| Trạng thái | Ý nghĩa                   |
| ---------- | ------------------------- |
| `ACTIVE`   | Được phép sử dụng         |
| `EXPIRED`  | Đã hết hạn                |
| `REVOKED`  | Bị thu hồi trước thời hạn |

Trước khi chạy, Launcher phải request execution lease.

MVP chỉ authorize customer và đúng Product Release. Customer được phép chạy trên
bất kỳ số máy hoặc Android device nào; Backend không nhận device ID/count, không
giới hạn active lease và không áp dụng concurrent-execution policy. Giới hạn tài
nguyên cục bộ, nếu có, thuộc Runtime scheduler và không phải license rule.

Request authorization gồm:

```json
{
  "productId": "product_01",
  "releaseId": "release_01",
  "packageHash": "sha256:...",
  "runtimeSessionId": "runtime_session_01",
  "runtimePublicKey": "base64-spki...",
  "runtimeKeyThumbprint": "sha256:..."
}
```

Server kiểm tra:

- Entitlement còn hiệu lực.
- Product không bị suspend.
- Release vẫn được phép chạy.
- Customer không bị ban.
- Package hash khớp immutable Product Release.
- Runtime public key và thumbprint hợp lệ.

Backend trả signed execution lease và content-key envelope được mã hóa riêng cho
Runtime session. Launcher giữ và chuyển chúng trong RAM nhưng không nhận plaintext
workflow key. Runtime verify Backend signature, session binding, package hash và
envelope trước khi giải mã Runtime IR trong memory.

Lease có `refreshAfter` và `expiresAt`. Launcher renew từ `refreshAfter`; Runtime
tiếp tục chạy bằng lease hiện tại cho đến hard expiry. Renew thành công trước
expiry chỉ thay lease, không cần giải mã lại workflow.

Nếu hard expiry xảy ra, Runtime không bắt đầu node mới, commit safe checkpoint,
chuyển execution sang `PAUSED` với reason `AUTHORIZATION_EXPIRED`, rồi xóa
plaintext workflow và transient key material. Resume yêu cầu Runtime session key
pair, signed lease và key envelope mới.

Contract đầy đủ nằm tại
[Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).

## 11. Revoke và suspend

| Hành động            | Phạm vi                                   |
| -------------------- | ----------------------------------------- |
| Revoke entitlement   | Chặn một customer dùng một product        |
| Disable release      | Chặn một release cụ thể                   |
| Suspend product      | Chặn toàn bộ product                      |
| Ban user             | Chặn toàn bộ hoạt động tài khoản          |

Khi revoke hoặc suspend trong MVP:

- Không cấp lease mới.
- Không renew lease hiện tại.
- Package có thể vẫn còn trên máy customer.
- Execution hiện tại được chạy đến khi lease hết hạn.
- Runtime pause trước khi bắt đầu node tiếp theo.
- Runtime xóa decrypted workflow và transient keys tại authorization boundary.
- Customer vẫn xem được order history.
- Admin phải nhập lý do và hệ thống ghi audit log.

Không hỗ trợ dừng execution ngay lập tức trong MVP vì Runtime không duy trì heartbeat hoặc command channel liên tục với Backend.

## 12. Update sản phẩm

Publisher không được ghi đè release đã publish.

```text
Publisher sửa project trong Studio
→ Tạo submission mới
→ Automated scan và admin review
→ Build ProductRelease mới
→ Đặt làm current release
```

Customer có thể:

- Tiếp tục dùng release cũ nếu vẫn được hỗ trợ.
- Update sang current release.
- Bị yêu cầu update nếu release cũ bị disable.

MVP chỉ cần:

- Một current release cho mỗi product.
- Giữ release cũ.
- Admin có thể disable release lỗi.
- Launcher kiểm tra minimum version và release status trước khi tạo run.

Chưa cần thiết kế rollback tự động phức tạp.
