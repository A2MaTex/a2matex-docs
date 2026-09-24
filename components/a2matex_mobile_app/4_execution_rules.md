# 10. Runtime có dịch Command thành JavaScript không?

Không.

Luồng thực tế:

```text
Server Execution IR step
→ Android command
→ Flutter Command Router
→ Native Command Handler
→ Android API
```

Ví dụ:

```text
ANDROID_TAP_ELEMENT IR
→ TAP_ELEMENT command
→ TapElementHandler
→ AccessibilityNodeInfo.performAction(ACTION_CLICK)
```

Runtime Agent không cần:

- JavaScript engine.
- Workflow Compiler.
- Graph interpreter.
- Node/edge parser.
- Flutter code động.
- APK riêng cho từng product.

Flutter chỉ là framework dùng để build A2MaTex App. Nó không phải ngôn ngữ để “dịch” command trong runtime.

# 11. Foreground Service và chạy nền

Khi người dùng bấm Run:

- App hiển thị foreground notification.
- Foreground service giữ execution session hoạt động.
- Flutter duy trì gRPC stream.
- Accessibility Service thực hiện action trong target app.
- Người dùng chuyển khỏi A2MaTex App sang target app.

MVP chấp nhận happy path:

```text
App process còn sống
+ gRPC stream còn kết nối
+ Accessibility Service còn bật
= workflow tiếp tục
```

Nếu process bị hệ điều hành kill hoặc stream mất:

- Runtime Agent dừng.
- Execution chuyển `FAILED`.
- Không resume.
- Người dùng chạy lại từ đầu.

Về lâu dài có thể chuyển gRPC coordinator xuống native foreground service, nhưng việc này sẽ tăng đáng kể Kotlin code và chưa cần cho MVP.
