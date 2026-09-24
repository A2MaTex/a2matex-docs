# 6. Có thể bấm, vuốt và điền input không?

Có, chủ yếu thông qua `AccessibilityService`.

## Click

```text
Tìm AccessibilityNodeInfo
→ Kiểm tra ACTION_CLICK
→ performAction(ACTION_CLICK)
```

Nếu node con không clickable, Runtime Agent có thể tìm parent gần nhất có `clickable = true`.

## Input text

```text
Tìm editable node
→ Focus node
→ ACTION_SET_TEXT
→ Kiểm tra text sau khi nhập
```

Có thể dùng clipboard/paste làm fallback, nhưng không nên là phương án mặc định vì liên quan dữ liệu nhạy cảm.

## Swipe

Sử dụng gesture được tạo từ:

- Điểm bắt đầu.
- Điểm kết thúc.
- Duration.

`AccessibilityService` hỗ trợ dispatch gesture tới màn hình nếu service khai báo capability phù hợp. ([API reference][3])

## Back và Home

Dùng global accessibility action:

```text
GLOBAL_ACTION_BACK
GLOBAL_ACTION_HOME
```

## Wait for element

Không chỉ `sleep` một khoảng cố định.

Runtime Agent nên:

1. Query accessibility tree.
2. Nếu chưa thấy, chờ accessibility event.
3. Query lại.
4. Dừng khi tìm thấy hoặc hết timeout.

# 7. Có cần thao tác theo tọa độ không?

Không nên dùng tọa độ làm phương thức chính.

Thứ tự selector nên là:

| Ưu tiên | Selector                            | Độ ổn định |
| ------: | ----------------------------------- | ---------- |
|       1 | `resourceId`                        | Cao nhất   |
|       2 | `contentDescription`                | Cao        |
|       3 | `text`                              | Trung bình |
|       4 | `className` kết hợp thuộc tính khác | Trung bình |
|       5 | Cây hierarchy/path/index            | Thấp       |
|       6 | Bounds hoặc tọa độ                  | Thấp nhất  |

Ví dụ:

```json
{
  "resourceId": "com.example:id/loginButton",
  "text": "Login",
  "contentDescription": "Login to account",
  "className": "android.widget.Button"
}
```

Runtime Agent có thể thử lần lượt các selector cho đến khi tìm thấy một node duy nhất.

# 8. Android UI có component ID hoặc XPath không?

Android không cung cấp XPath chuẩn như trình duyệt.

Thay vào đó, Accessibility Service có thể đọc một cây `AccessibilityNodeInfo`.

Một node có thể cung cấp:

- Package name.
- Class name.
- View resource ID.
- Text.
- Content description.
- Clickable.
- Editable.
- Scrollable.
- Enabled.
- Bounds trên màn hình.
- Các accessibility action được hỗ trợ.
- Quan hệ parent/children.

Android cho phép Accessibility Service lấy active window và cây `AccessibilityNodeInfo`; service cũng có thể giới hạn package cần theo dõi. ([API reference][3])

A2MaTex có thể xây selector path riêng:

```text
Window
→ RecyclerView[class]
→ LinearLayout[index=2]
→ Button[text="Login"]
```

Nhưng path/index không ổn định như XPath của một DOM cố định.

## Khi nào không tìm được element?

Không phải app nào cũng expose accessibility tree đầy đủ.

Các trường hợp khó:

- Game render bằng OpenGL.
- Canvas custom.
- Custom View không khai báo accessibility.
- Một số WebView.
- UI bị obfuscate hoặc cố tình chặn accessibility.
- Element chưa được render.
- RecyclerView item chưa xuất hiện trên màn hình.
- Popup thuộc system window khác.
- UI thay đổi theo phiên bản app.

Trong các trường hợp này có thể cần:

- Scroll để element được render.
- Text/content-description fallback.
- Coordinate fallback.
- OCR hoặc image matching trong tương lai.

MVP chưa nên hỗ trợ OCR hoặc image matching.

# 9. Có thể click element chưa nhìn thấy không?

Chỉ khi element đã tồn tại trong accessibility tree và hỗ trợ action.

Nếu element chưa được render vì nằm sâu trong danh sách:

```text
Tìm scrollable parent
→ ACTION_SCROLL_FORWARD
→ Chờ UI cập nhật
→ Query lại
→ Click khi tìm thấy
```

Không thể đảm bảo click trực tiếp một component:

- Chưa được render.
- Không có trong active window.
- Không xuất hiện trong accessibility tree.

Vì vậy node `TapElement` có thể có cấu hình:

```json
{
  "selector": {
    "resourceId": "com.example:id/submit"
  },
  "autoScroll": true,
  "maxScrollAttempts": 5,
  "timeoutMs": 10000,
  "allowCoordinateFallback": false
}
```
