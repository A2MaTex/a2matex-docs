# 13. Entitlement thay cho License Key

Không cần phát license key dài để customer nhập thủ công.

Quyền sử dụng được biểu diễn bằng một `ProductEntitlement` trên server:

```json
{
  "id": "entitlement_01J...",
  "userId": "user_01J...",
  "productId": "product_01J...",
  "orderId": "order_01J...",
  "status": "ACTIVE",
  "startsAt": "2026-09-23T00:00:00Z",
  "expiresAt": "2026-10-23T00:00:00Z",
  "maxDevices": 3,
  "maxConcurrentExecutions": 1
}
```

MVP chỉ cần ba trạng thái:

| Status    | Ý nghĩa                             |
| --------- | ----------------------------------- |
| `ACTIVE`  | Được phép sử dụng                   |
| `EXPIRED` | Hết thời hạn                        |
| `REVOKED` | Bị thu hồi thủ công hoặc do vi phạm |

Customer đăng nhập A2MaTex App bằng account. Server kiểm tra entitlement trực tiếp, không tin license data do client gửi.

# 14. Kiểm tra quyền khi Customer Launch

Khi customer bấm `Run`, server kiểm tra theo thứ tự:

```text
User ACTIVE?
→ Device active?
→ Product ACTIVE?
→ Published release tồn tại?
→ Entitlement ACTIVE?
→ Chưa hết hạn?
→ Device quota còn?
→ Concurrent execution quota còn?
```

Nếu hợp lệ:

1. Server tạo `Execution`.
2. Chọn `Product.currentReleaseId`.
3. Sinh stream token ngắn hạn.
4. Trả `executionId` và stream token.
5. A2MaTex App mở gRPC stream.
6. Server load Execution IR.
7. Workflow Engine bắt đầu chạy.
8. Android chỉ nhận từng command.

Customer không tải xuống:

- Source graph.
- JavaScript source.
- Execution IR.
- Script artifact.
- Toàn bộ product logic.

# 15. Multi-device và hệ thống máy ảo

Một entitlement có thể cho phép nhiều device:

```text
maxDevices = 10
maxConcurrentExecutions = 10
```

Mỗi máy thật hoặc máy ảo phải:

- Đăng ký thành một Device riêng.
- Có `deviceId` riêng.
- Mở gRPC stream riêng.
- Có `executionId` riêng.
- Có command sequence riêng.

Không sử dụng:

```text
Một stream
→ Client tự replicate command sang 10 máy
```

Sử dụng:

```text
Một entitlement
→ 10 device
→ 10 execution
→ 10 gRPC stream
```

Marketplace chỉ quản lý quota. Việc fan-out execution được xử lý ở runtime/server, không phải frontend.

# 16. Các mức thu hồi quyền

Không nên dùng một cơ chế revoke cho mọi trường hợp.

| Mức vi phạm                  | Hành động                     |
| ---------------------------- | ----------------------------- |
| Một thiết bị đáng ngờ        | Disable `Device`              |
| Vi phạm với một sản phẩm     | Revoke `ProductEntitlement`   |
| Account vi phạm nghiêm trọng | Chuyển `User` thành `BANNED`  |
| Product có lỗ hổng           | Suspend `Product`             |
| Một release bị lỗi           | Rollback hoặc disable release |

Điều này phù hợp với các bảng hiện có:

- `User.status`.
- `Device.isActive`.
- `ProductEntitlement.status`.
- `Product.status`.
- `ProductRelease`.

# 17. Revoke giữa lúc đang chạy

Khi admin revoke entitlement:

```text
Admin chọn Revoke
→ Entitlement chuyển REVOKED
→ Xóa entitlement cache
→ Tìm active executions
→ Đánh dấu execution bị revoke
→ Thông báo Runtime Connection Manager
→ Gửi execution revoked
→ Đóng gRPC stream
→ Không gửi command tiếp theo
```

Customer không thể tiếp tục chạy offline vì:

- Product logic vẫn nằm trên server.
- Android chỉ có command hiện tại.
- Mất stream thì execution dừng.
- Stream token không còn hợp lệ.
- Start execution mới bị entitlement check từ chối.

Nếu hệ thống đang có nhiều NestJS instance, revoke event có thể truyền qua Redis Pub/Sub. MVP chạy một instance có thể đóng stream trực tiếp bằng connection map trong memory.

# 18. Phân biệt Revoke và Suspend

## Revoke Entitlement

Ảnh hưởng một customer:

```text
Customer A không dùng được Product X
Customer B vẫn dùng bình thường
```

## Disable Device

Ảnh hưởng một máy:

```text
Device A bị khóa
Các device khác của customer vẫn hoạt động
```

## Ban User

Ảnh hưởng toàn bộ account:

```text
Không login
Không mua
Không chạy bất kỳ product nào
```

## Suspend Product

Ảnh hưởng tất cả customer:

```text
Không tạo execution mới
Có thể dừng execution hiện tại
Product bị ẩn khỏi marketplace
```
