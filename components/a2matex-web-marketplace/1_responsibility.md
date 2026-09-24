## 1. Ranh giới giữa Studio và Marketplace

### A2MaTex Studio

Studio chịu trách nhiệm:

- Thiết kế workflow.
- Validate graph và JavaScript.
- Compile thử thành IR.
- Local debug.
- Chuẩn bị submission snapshot.
- Gửi submission lên server.

### Web Marketplace

Marketplace chịu trách nhiệm:

- Hiển thị submission cho publisher.
- Cung cấp công cụ review cho admin.
- Quản lý product listing.
- Quản lý phiên bản đã publish.
- Bán license.
- Quản lý quyền sử dụng.
- Suspend sản phẩm.
- Revoke license, device hoặc account.

Marketplace không:

- Chỉnh sửa graph.
- Compile workflow ở frontend.
- Chạy Android command.
- Gửi toàn bộ workflow cho customer.
- Build APK hoặc executable riêng cho từng sản phẩm.
