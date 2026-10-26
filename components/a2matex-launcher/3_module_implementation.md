# 3. Các module cần triển khai

```text
Launcher
├── App Shell
├── Authentication
├── Product Library
├── Package Manager
├── Runtime Supervisor
├── Runtime Client
├── Device Manager
├── Input Profile Manager
├── Credential Manager
├── Run Configuration
├── Execution Monitor
├── License Lease Manager
├── Local Settings
├── Background/Tray Manager
└── Launcher Updater
```

## 3.1. Authentication

Chịu trách nhiệm:

- đăng nhập;
- refresh token;
- đăng xuất;
- lấy thông tin user;
- kiểm tra role/customer status;
- xử lý session hết hạn.

Token không lưu trong `localStorage`. Main Process lưu bằng Windows Credential Manager hoặc cơ chế mã hóa dựa trên Windows DPAPI.

## 3.2. Product Library

Hiển thị sản phẩm customer có quyền sử dụng:

```text
My Products
├── Installed
├── Update Available
├── Not Installed
├── License Expired
└── Revoked
```

Thông tin sản phẩm:

- product name;
- publisher;
- installed release;
- latest release;
- license expiration;
- capability yêu cầu;
- Runtime version tối thiểu;
- hướng dẫn sử dụng.

Việc mua và khám phá sản phẩm có thể thực hiện trên Marketplace web. Launcher tập trung vào “My Library”.

## 3.3. Package Manager

Package Manager quản lý workflow package đã tải:

```text
%LOCALAPPDATA%\A2MaTeX\packages\
└── <productId>\
    └── <releaseId>\
        ├── package.enc
        └── manifest.json
```

Trạng thái package:

```text
NOT_INSTALLED
→ DOWNLOADING
→ VERIFYING
→ INSTALLED
→ UPDATE_AVAILABLE
→ CORRUPTED
→ REVOKED
```

Quy trình cài:

1. Xin download URL từ backend.
2. Tải package vào temporary file.
3. Kiểm tra size và hash.
4. Đọc manifest.
5. Kiểm tra Runtime compatibility.
6. Chuyển file vào release directory.
7. Đánh dấu installed.

Update phải atomic:

```text
Release cũ vẫn hoạt động
→ tải release mới
→ verify thành công
→ activate release mới
```

Nếu update lỗi, giữ release hiện tại.

Runtime vẫn phải kiểm tra hash/signature lại lúc chạy; không tin hoàn toàn kết quả verify từ Launcher.

## 3.4. Runtime Supervisor

Runtime Supervisor quản lý process `a2matex-runtime-agent.exe`.

Khi Launcher khởi động:

1. Kiểm tra Agent có đang chạy không.
2. Nếu chưa chạy thì khởi động Agent.
3. Chờ health check.
4. Thực hiện handshake.
5. Kiểm tra Runtime version và supported handlers.
6. Mở event connection.

Handshake:

```json
{
  "runtimeVersion": "0.1.0",
  "mode": "production",
  "supportedHandlers": [
    {
      "type": "android.tap",
      "version": 1
    },
    {
      "type": "flow.if",
      "version": 1
    }
  ]
}
```

Các trạng thái:

```text
NOT_FOUND
STARTING
CONNECTED
INCOMPATIBLE
UPDATING
DISCONNECTED
CRASHED
```

Nếu Agent crash, Launcher:

- thông báo cho người dùng;
- đánh dấu các run chưa kết thúc là interrupted;
- không tự restart và resume workflow một cách mù quáng;
- có thể restart Agent để đọc checkpoint.

## 3.5. Runtime Client

Runtime Client gọi local API:

```text
GET  /v1/health
GET  /v1/runtime-info
GET  /v1/devices
POST /v1/devices/refresh

POST /v1/runs
GET  /v1/runs/{id}
POST /v1/runs/{id}/pause
POST /v1/runs/{id}/resume
POST /v1/runs/{id}/cancel

GET  /v1/events
```

Launcher không gọi `ldconsole list2` hoặc `adb devices`. Nó chỉ gọi:

```text
Runtime Agent → Device Registry → Launcher
```

## 3.6. Device Manager

Hiển thị danh sách thiết bị Runtime phát hiện:

| Thiết bị   | Trạng thái | Độ phân giải | Execution   |
| ---------- | ---------- | -----------: | ----------- |
| LDPlayer-0 | Online     |    1080×1920 | Available   |
| LDPlayer-1 | Booting    |            — | Unavailable |
| LDPlayer-2 | Online     |     720×1280 | Running     |
| LDPlayer-3 | Offline    |            — | Unavailable |

Các chức năng:

- refresh danh sách;
- lọc online/offline;
- chọn nhiều thiết bị;
- đặt tên hiển thị;
- xem LDPlayer index;
- kiểm tra kết nối;
- mở hoặc dừng instance thông qua Runtime;
- xem thiết bị đang chạy workflow nào.

Launcher chỉ cho chọn thiết bị:

```text
ONLINE + ADB_READY + NOT_BUSY
```

## 3.7. Input Profile Manager

Workflow release khai báo input schema:

```json
{
  "inputs": [
    {
      "key": "accountName",
      "type": "string",
      "required": true
    },
    {
      "key": "sheetCredential",
      "type": "credential_ref",
      "required": true
    }
  ]
}
```

Launcher tự sinh form từ schema.

Customer có thể lưu profile:

```text
Account A
├── username
├── non-secret config
└── credential reference

Account B
├── username
├── non-secret config
└── credential reference
```

Sau đó gán profile cho thiết bị:

```text
LDPlayer-0 → Account A
LDPlayer-1 → Account B
LDPlayer-2 → Account C
```

Mỗi `DeviceExecution` nhận input riêng.

## 3.8. Credential Manager

Credential không được lưu trong workflow graph.

Launcher lưu:

```text
credential_ref → encrypted credential
```

Ví dụ:

```json
{
  "credentialRef": "cred_google_account_a",
  "type": "GOOGLE_OAUTH"
}
```

Trong run request chỉ gửi reference hoặc credential tạm thời đã được bảo vệ:

```json
{
  "sheetCredentialRef": "cred_google_account_a"
}
```

MVP có thể để backend quản lý OAuth connector. Với local connector, Launcher chỉ truyền secret cho Runtime trong memory và Runtime không ghi xuống SQLite/log.

## 3.9. Run Configuration

Trước khi Run, customer cấu hình:

- product/release;
- danh sách thiết bị;
- input profile từng thiết bị;
- optional local parallel-device preference;
- failure policy;
- connector credential;
- optional run name.

Ví dụ:

```json
{
  "releaseId": "release_01",
  "maxParallelDevices": 3,
  "failurePolicy": "CONTINUE_OTHER_DEVICES",
  "devices": [
    {
      "deviceId": "ldplayer:0",
      "profileId": "profile_a"
    },
    {
      "deviceId": "ldplayer:1",
      "profileId": "profile_b"
    }
  ]
}
```

Launcher thực hiện preflight:

- package đã cài;
- license còn hiệu lực;
- Runtime tương thích;
- thiết bị online;
- input đầy đủ;
- credential tồn tại;
- Runtime còn đủ local capacity hoặc có thể queue execution.

`maxParallelDevices` chỉ là local scheduling preference theo tài nguyên máy. Nó
không được gửi lên Backend và không giới hạn quyền sử dụng Product.

## 3.10. Production Authorization Manager

Launcher owns customer-authenticated Backend calls but never decrypts Product
workflow content. The authoritative protocol is
[Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).

When customer clicks **Run**, Launcher:

1. asks Runtime to prepare a Product Run authorization session;
2. receives `runtimeSessionId`, Runtime public key, algorithm, and thumbprint;
3. requests Backend authorization for the exact Product, Release, package hash,
   and Runtime session;
4. receives a signed execution lease and Runtime-bound key envelope;
5. forwards the encrypted package reference, lease, envelope, selected devices,
   and inputs to Runtime.

Launcher never receives a plaintext workflow key. It keeps the signed lease and
key envelope in memory only and never places either value in files, local
storage, telemetry, or logs.

### Renewal

Runtime emits `AUTHORIZATION_RENEWAL_REQUIRED` at the signed lease's
`refreshAfter`. Launcher renews with Backend while the old lease remains valid,
then forwards the newer signed lease to Runtime. Runtime verifies the signature,
lease family, increasing sequence, Product/Release/package, and Runtime-session
binding before atomically replacing the old lease.

Launcher retries only transient connectivity, timeout, `429`, and `5xx`
responses with bounded exponential backoff and jitter. It reuses one request ID
and stops at the current `expiresAt`. Authorization denial, revoked entitlement,
disabled Release, package mismatch, and invalid session binding require an
explicit error state rather than blind retry.

Launcher stays available in its background/system-tray process while a Product
Run needs renewal. If it exits completely, Runtime continues only until the
current lease expires.

### Hard expiry and reauthorization

At hard expiry Runtime stops admitting new node attempts, checkpoints the last
unambiguous result, enters `PAUSED` with reason `AUTHORIZATION_EXPIRED`, and
removes decrypted workflow/key material. Launcher must then obtain a new Runtime
session, signed lease, and key envelope. Runtime revalidates and decrypts the
same package before resuming from the last safe checkpoint.

## 3.11. Execution Monitor

Hiển thị hai cấp.

### Run group

```text
Running:   3
Queued:    2
Succeeded: 4
Failed:    1
```

### Device execution

```text
LDPlayer-0
Status: RUNNING
Current node: Wait For Element
Sequence: 17
Loop iteration: 3
Duration: 02:14
```

Cho phép xem:

- current node;
- node history;
- input/output;
- error;
- screenshot artifact;
- retry count;
- logs;
- execution duration.

Dữ liệu này lấy từ Runtime Event Stream, không do Launcher tự suy luận.
