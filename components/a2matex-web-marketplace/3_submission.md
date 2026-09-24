# 3. Submission từ Studio gửi lên

Submission là một snapshot bất biến của cloud draft tại một revision cụ thể.

```text
Submission
├── Source snapshot
│   ├── Project graph
│   ├── JavaScript source
│   ├── Variables và input schema
│   └── Asset references
│
├── Product manifest
│   ├── Version
│   ├── Required capabilities
│   ├── Target packages
│   ├── Runtime limits
│   └── Minimum Runtime version
│
├── Marketplace metadata
│   ├── Name
│   ├── Description
│   ├── Icon
│   ├── Screenshots
│   ├── Category
│   └── Release notes
│
└── Test information
    ├── Last test result
    ├── Test device
    ├── Android version
    └── Studio/compiler version
```

Một submission phải gắn với:

```text
productId
publisherId
projectId
draftRevision
version
sourceHash
```

Sau khi submit:

- Submission không được chỉnh sửa.
- Publisher muốn thay đổi phải tạo submission mới.
- Cloud draft vẫn được tiếp tục chỉnh sửa độc lập.
- Admin luôn review đúng snapshot đã gửi.

# 4. Trạng thái Submission

Để giữ MVP đơn giản, chỉ cần bốn trạng thái:

| Status              | Ý nghĩa                                      |
| ------------------- | -------------------------------------------- |
| `PENDING`           | Đang chờ automated check hoặc admin review   |
| `CHANGES_REQUESTED` | Admin yêu cầu publisher chỉnh sửa và gửi lại |
| `APPROVED`          | Submission đã được duyệt                     |
| `REJECTED`          | Submission bị từ chối                        |

Không cần thêm `IN_REVIEW`, `BUILDING`, `READY_TO_PUBLISH` trong MVP. Tiến trình kiểm tra chi tiết có thể thể hiện bằng log thay vì thêm status.
