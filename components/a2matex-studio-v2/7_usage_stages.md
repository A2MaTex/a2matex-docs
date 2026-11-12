
# 11. Quy trình sử dụng Studio

## 11.1. Tạo project

Người dùng chọn:

- Blank Automation
- Android Interaction Template
- Monitor and Notify Template
- Collect Data to Sheet Template

Sau đó nhập:

- tên sản phẩm
- mô tả
- package Android mục tiêu nếu có
- Runtime version
- input mà customer cần cung cấp

## 11.2. Chọn thiết bị test

Studio gọi Runtime Agent để lấy:

- thiết bị Android thật
- ADB serial
- LDPlayer instance
- độ phân giải
- trạng thái online

Người dùng chọn một thiết bị mặc định.

## 11.3. Thiết kế graph

Người dùng:

1. Kéo `Start`.
2. Thêm `Launch App`.
3. Thêm `Wait For Element`.
4. Tạo nhánh `found` và `timeout`.
5. Thêm `Tap`.
6. Thêm `Repeat`.
7. Thêm `Google Sheets`.
8. Kết thúc bằng `End`.

## 11.4. Cấu hình node

Ví dụ với Tap:

```text
Target mode: Output from another node
Source: Wait For Element
Coordinate: output.center
Timeout: 5 seconds
On error: Retry 2 times, then follow error branch
```

## 11.5. Chạy node đơn lẻ

Studio cho phép:

- chạy node đã chọn;
- dùng input được pin từ lần chạy trước;
- dùng mock input;
- kiểm tra ADB command/result;
- xem screenshot trước và sau.

Đây là tính năng rất quan trọng để UAT nhanh. Người dùng không phải chạy lại toàn graph chỉ để chỉnh một tọa độ.

## 11.6. Chạy từ một node

Người dùng có thể chọn `Run from here`, với điều kiện:

- các input cần thiết đã được pin hoặc cung cấp;
- device state phù hợp;
- Studio cảnh báo đây không hoàn toàn giống chạy từ Start.

## 11.7. Chạy toàn workflow

Runtime Agent tạo một development execution. Canvas hiển thị:

- node đang chạy;
- node thành công;
- node lỗi;
- nhánh được chọn;
- loop iteration;
- execution duration.

Studio có thể hỗ trợ nhiều test profile:

```text
Profile A → device A → input A
Profile B → device B → input B
```

MVP mặc định test một thiết bị để dễ sử dụng, nhưng nên có chế độ **multi-device smoke test** để xác nhận các thiết bị có thể rẽ nhánh độc lập.

## 11.8. Publish

Trước khi submit:

1. Flush autosave và lưu Project Draft vào SQLite.
2. Sync Project Draft shape hiện tại lên backend.
3. Hiển thị publish checklist mang tính authoring.
4. Gửi yêu cầu tạo Submission cho Product và Project Draft đã sync.
5. Backend khóa exact draft, tạo Workflow Revision pending và chạy background scan.
6. Backend validate, compile authoritative Runtime IR và derive capability manifest.
7. Studio chỉ hiển thị trạng thái Submission được phép hiển thị cho Publisher.

Runtime IR trong Project Draft, nếu có, là cache do local Runtime compile API trả về.
Backend lưu cùng draft payload để giữ Project Draft shape đồng nhất, nhưng Submission
không tin hoặc copy cache này.


# 12. Local project, draft và revision

Cần phân biệt ba loại dữ liệu:

| Loại | Có thể sửa? | Vị trí |
|---|---:|---|
| Local project | Có | Máy publisher |
| Cloud draft | Có | Backend |
| Workflow revision | Không | Backend |
| Published release | Không | Marketplace |

Studio có autosave local và crash recovery. Publisher có thể đồng bộ draft lên server, nhưng lần submit phải tạo một revision bất biến.

Workflow Revision do Backend tạo và hoàn thiện sau scan, bao gồm:

- editor graph;
- input schema;
- Runtime IR;
- node version lock;
- capability manifest;
- assets;
- product metadata.

Marketplace review đúng revision đó, không review “product hiện tại” vốn còn có thể thay đổi.
