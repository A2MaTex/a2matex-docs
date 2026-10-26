# 4. Luồng người dùng

## 4.1. Lần đầu mở Launcher

```text
Mở Launcher
→ đăng nhập
→ kiểm tra Runtime Agent
→ tìm LDPlayer installation
→ Runtime discovery devices
→ đồng bộ My Products
→ hiển thị Dashboard
```

Nếu không tìm thấy LDPlayer:

```text
Không tìm thấy LDPlayer
→ customer chọn thư mục cài đặt
→ Launcher gửi config cho Runtime
→ Runtime validate ldconsole.exe/adb.exe
→ refresh devices
```

## 4.2. Cài sản phẩm

```text
My Products
→ chọn sản phẩm
→ chọn Install
→ tải package
→ verify
→ kiểm tra Runtime compatibility
→ installed
```

## 4.3. Chạy workflow

```mermaid
sequenceDiagram
    participant U as Customer
    participant L as Launcher
    participant B as Backend
    participant R as Runtime Agent

    U->>L: Chọn sản phẩm, thiết bị và Run
    L->>L: Validate input/profile
    L->>R: Prepare Product Run authorization session
    R-->>L: Session ID, public key, thumbprint
    L->>B: Request authorization for Release/package/session
    B-->>L: Signed lease + Runtime-bound key envelope
    L->>R: Start encrypted Product Run
    R-->>L: Run group created
    R-->>L: Execution events
    R-->>L: AUTHORIZATION_RENEWAL_REQUIRED
    L->>B: Renew signed lease
    B-->>L: New lease with later expiry
    L->>R: Submit renewed lease
    R-->>L: Final results
```

Start request:

```json
{
  "release": {
    "productId": "product_01",
    "releaseId": "release_03",
    "packagePath": "packages/product_01/release_03/package.enc"
  },
  "lease": "signed-lease",
  "keyEnvelope": {},
  "devicePlans": [
    {
      "deviceId": "ldplayer:0",
      "inputs": {
        "accountName": "Account A"
      }
    }
  ]
}
```

Backend authorization is independent of device count and machine count.
Runtime may queue selected devices according to local hardware capacity. The
signed lease and key envelope follow
[Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).

Runtime trả:

```json
{
  "runGroupId": "run_01",
  "executions": [
    {
      "executionId": "exec_01",
      "deviceId": "ldplayer:0",
      "status": "QUEUED"
    }
  ]
}
```
