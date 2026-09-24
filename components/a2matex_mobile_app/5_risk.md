# 12. Nguy cơ bị dịch ngược

Không thể ngăn người sở hữu thiết bị reverse-engineer hoàn toàn client app.

Họ có thể tìm thấy:

- Danh sách command handler.
- Protobuf message.
- Cách selector engine hoạt động.
- Cơ chế gọi Accessibility API.
- Runtime validation logic.
- Endpoint server.

Họ không nhận được:

- Full workflow graph.
- JavaScript product.
- Execution IR.
- Branch logic.
- Các command tương lai.
- Connector credentials.
- Logic của publisher.
- Server-side license rules.

Ngay cả khi quan sát command hiện tại, họ chỉ biết:

```text
Bây giờ hãy bấm Login
```

Họ không biết toàn bộ workflow phía sau:

```text
Nếu đăng nhập thành công
→ đọc số dư
→ chạy JavaScript
→ kiểm tra điều kiện
→ gửi Telegram
→ thực hiện bước khác
```

Lớp bảo vệ chính là:

- Logic nằm trên server.
- Một command tại một thời điểm.
- TLS.
- Stream token ngắn hạn.
- Bind token với device và execution.
- Sequence và expiration.
- Handler allowlist.
- Không có command thực thi code.
- Không gửi IR xuống mobile.

Obfuscation chỉ làm reverse engineering khó hơn, không phải security boundary.

# 13. Rủi ro Google Play

Đây là rủi ro sản phẩm cần chốt sớm.

Google Play cho phép deterministic, rule-based automation với mục đích rõ ràng, nhưng cấm việc Accessibility API tự chủ động lập kế hoạch, ra quyết định và thực thi hành động một cách không giới hạn. App không phải accessibility tool cũng phải khai báo, giải thích và hiển thị disclosure/consent phù hợp. ([Play Console Help](https://support.google.com/googleplay/android-developer/answer/10964491?hl=en))

A2MaTex cần:

- Workflow là script tĩnh do con người tạo.
- Hiển thị rõ product sẽ thao tác ứng dụng nào.
- Hiển thị capability trước khi chạy.
- Yêu cầu customer chủ động bấm Run.
- Có nút Stop luôn khả dụng.
- Không tự kích hoạt workflow bí mật.
- Không thao tác ngoài target package đã khai báo.
- Không giả mạo mình là accessibility tool nếu mục tiêu chính không hỗ trợ người khuyết tật.
- Chuẩn bị Play Console Accessibility declaration.

Việc publish qua Google Play vẫn có nguy cơ bị từ chối tùy cách triển khai và use case. Phương án sideload APK hoặc enterprise distribution có thể là fallback, nhưng không nên mặc định ngay từ MVP.
