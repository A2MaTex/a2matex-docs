A2MaTeX Launcher là ứng dụng Windows dành cho customer. Nó là lớp giao diện và điều phối bên ngoài Runtime Agent:

```text
Customer
→ Launcher
→ Runtime Agent
→ LDPlayer/Android devices
```

Launcher không tự chạy workflow và không trực tiếp gọi ADB. Nó quản lý sản phẩm, license, cấu hình chạy, thiết bị được chọn và hiển thị kết quả.

# 1. Trách nhiệm của Launcher

Launcher chịu trách nhiệm:

- Đăng nhập tài khoản A2MaTeX.
- Đồng bộ các sản phẩm customer đã mua.
- Tải và cập nhật workflow release.
- Quản lý package đã cài trên máy.
- Kiểm tra Runtime Agent và version tương thích.
- Hiển thị danh sách LDPlayer instance do Runtime phát hiện.
- Cho phép chọn thiết bị chạy workflow.
- Gán input, account và credential cho từng thiết bị.
- Chuyển Runtime session public key lên Backend và xin signed execution lease cùng key envelope.
- Gửi yêu cầu bắt đầu run group tới Runtime.
- Theo dõi trạng thái từng device execution.
- Pause, resume và cancel.
- Giữ ứng dụng chạy background/system tray.
- Gia hạn execution lease từ `refreshAfter` trong thời gian chạy.
- Gửi run summary về backend nếu cần.

Launcher không chịu trách nhiệm:

- Parse và thực thi graph.
- Chọn node tiếp theo.
- Tạo goroutine cho thiết bị.
- Gọi `adb.exe`, `ld.exe` hoặc `ldconsole.exe`.
- Lưu `currentNode`, sequence và variables làm source of truth.
- Review hoặc chỉnh sửa workflow.
- Giải quyết logic retry của từng node.
- Giải mã Product workflow hoặc nhận plaintext release content key.

MVP không giới hạn machine, Android device hoặc concurrent execution ở Backend.
Launcher có thể cho customer chọn nhiều device; Runtime tự queue/bound parallel
work theo tài nguyên local.
