
# 27. Package và license trong Runtime Agent

Có hai giai đoạn.

## Development

Studio gửi Runtime IR trực tiếp:

```json
{
  "workflow": {},
  "devices": [],
  "testInputs": {}
}
```

Runtime đánh dấu run là development.

## Production

Launcher cung cấp:

- encrypted release/package reference;
- signed execution lease;
- Runtime-session-bound key envelope;
- selected devices;
- device inputs;
- credential references.

Runtime Agent:

1. tạo ephemeral authorization-session key pair trong memory;
2. trả public key và thumbprint cho Launcher;
3. verify Backend lease signature, time window, release/package và session binding;
4. mở key envelope bằng Runtime private key;
5. kiểm tra package/payload hash và Runtime compatibility;
6. giải mã, validate và giữ Runtime IR trong memory;
7. enforce `refreshAfter`/`expiresAt` trước mỗi node attempt;
8. xóa plaintext workflow và transient keys khi kết thúc hoặc hard expiry.

Launcher không giải mã graph và không nhận plaintext content key trong normal
application flow. Backend không giới hạn installation, machine, device count
hoặc concurrent run; local Runtime vẫn bound parallel work vì resource safety.

Renewal trước expiry chỉ thay signed lease. Hard expiry dừng node admission,
checkpoint execution, chuyển sang `PAUSED` với reason
`AUTHORIZATION_EXPIRED`, xóa decrypted data/key material và yêu cầu session key
pair, lease và envelope mới để resume.

Contract authoritative:
[Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).
