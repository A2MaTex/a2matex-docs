# Permissions, Credentials, Entitlement, and License

## 1. Why two permission layers are required

Android grants Camera, Bluetooth, files, and notification permissions to the
Android Launcher APK. It does not grant them independently to each `.apkg`.

A2MaTeX therefore maintains:

1. **Android OS permission** for the Launcher.
2. **Product-level grant** for each installed product and capability.

Example:

```text
Android Settings
  A2MaTeX Launcher -> Camera: Allowed

A2MaTeX product grants
  Invoice Scanner  -> Camera: Allowed
  Bluetooth Exporter -> Camera: Denied
```

Both layers must allow an operation.

## 2. Capability Registry for UAT

| Capability | Meaning | Main checks |
|---|---|---|
| `network.request` | Make an HTTP request through the platform client | Domain, TLS, request/response size, count, timeout |
| `camera.capture` | Capture an image for the product action | Manifest, approval, product grant, Android Camera permission |
| `bluetooth.scan` | Scan nearby devices | Android version-specific permission, grant, timeout |
| `bluetooth.connect` | Connect to a selected device | Approved flow, selected device, grant |
| `file.user_selected.read` | Read a user-selected file | Scoped URI, type and size policy |
| `file.user_selected.write` | Write to a user-selected destination | Scoped destination, type and size policy |
| `notification.send` | Show a product-related notification | Notification permission/policy, rate limit |
| `credential.read_reference` | Use a bound credential | Approved slot, ownership, scopes, product binding |

The Launcher manifest must declare Android permissions needed by the fixed
Capability Registry. An `.apkg` cannot add a new Android permission after the
Launcher is installed.

## 3. Capability decision

A call is allowed only when all relevant conditions pass:

```text
Package declares capability
AND automated extraction agrees
AND Reviewer approved it
AND Product grant is Allowed
AND Android OS permission is Allowed, when applicable
AND requested domain/credential is permitted
AND License Lease is valid
AND runtime quota is available
```

Failure is returned as a stable error code such as:

- `PERMISSION_DENIED`.
- `PERMISSION_REVOKED`.
- `DOMAIN_NOT_ALLOWED`.
- `CREDENTIAL_MISSING`.
- `LICENSE_REQUIRED`.
- `RESOURCE_LIMIT`.

## 4. Permission disclosure

Before the first relevant run, Launcher should show purpose rather than only
technical permission names:

```text
Invoice Scanner wants to:

- Use the camera when you capture an invoice.
- Read an image you select.
- Send data to sheets.googleapis.com.
- Use your connected Google Account.

[Allow and run] [Cancel]
```

Customer can later inspect:

```text
Invoice Scanner permissions

Camera              Allowed
Selected files      Allowed
Google Account      Connected
Network             sheets.googleapis.com

[Revoke camera]
[Disconnect Google]
[Revoke all access]
```

## 5. Revoking permission during execution

1. Product grant changes to `DENIED`.
2. Runtime receives a permission-change event.
3. Capability Broker cancels work that has not started.
4. Broker attempts to close an in-flight native call where the API supports it.
5. All later calls are rejected.
6. The node receives `PERMISSION_REVOKED`.
7. Workflow follows an error edge or stops according to policy.

If the Customer revokes the Android OS permission in Settings, every product
using that Android capability is affected. Launcher should offer **Open Android
Permission Settings** when appropriate.

## 6. Customer-owned credentials

Used when the workflow accesses a Customer resource:

- Google Sheets.
- Gmail or Calendar.
- Telegram account/bot owned by the Customer.
- Dropbox, Drive, or other Customer storage.

### Binding flow

1. Publisher declares a credential slot in the workflow/product.
2. Customer acquires and installs the product.
3. Launcher displays **Connect Account** for the slot.
4. Customer completes OAuth or provider-specific authorization.
5. Credential Vault stores token material.
6. Product stores only a credential reference.
7. Runtime injects or proxies credential use when an approved node runs.
8. Customer may disconnect or bind another account.

Graph stores:

```json
{
  "credentialBinding": {
    "type": "google-oauth",
    "slot": "customer-google-account"
  }
}
```

Graph and package never store:

```json
{
  "accessToken": "ya29..."
}
```

Publisher, Reviewer, Admin, and Support must not be able to view the raw Customer
token through normal product operations.

## 7. Publisher-owned credentials

Used when the Publisher owns or pays for a service:

- Publisher API.
- OCR service subscription.
- Publisher-managed Telegram bot.
- Publisher SaaS backend.

### UAT model: Publisher Backend

```text
Launcher/Runtime -> Publisher Backend -> Third-party API
```

The third-party secret remains on the Publisher Backend. The `.apkg` may contain
only the approved Publisher endpoint and request contract.

### Later model: Platform Credential Proxy

```text
Launcher/Runtime -> Platform Credential Proxy -> Third-party API
```

This later model requires encrypted secret storage, Publisher isolation, rate
limits, billing, abuse detection, and request signing. It is not required for
UAT.

## 8. Entitlement

Entitlement is the server-side business right to use a Product.

Conceptual fields:

```text
user/customer
product
plan
starts_at
expires_at
status
features
```

Possible status values:

- `PENDING`.
- `ACTIVE`.
- `SUSPENDED`.
- `EXPIRED`.
- `REVOKED`.
- `REFUNDED` after payment is introduced.

For UAT, an Admin may grant an active entitlement without a real payment flow.

## 9. License Lease

A License Lease is a signed, short-lived authorization that lets one device run
a Product Version without checking the server for every launch.

Conceptual payload:

```json
{
  "leaseId": "lease-123",
  "userId": "user-123",
  "productId": "product-456",
  "versionRange": ">=1.0.0 <2.0.0",
  "deviceId": "device-789",
  "issuedAt": 1790000000,
  "refreshAfter": 1790086400,
  "offlineUntil": 1790269200,
  "entitlementExpiresAt": 1821536000,
  "features": ["standard"],
  "signature": "..."
}
```

### Issue/refresh request checks

- User is authenticated.
- Account is not blocked.
- Entitlement exists and is active.
- Product and requested version are allowed.
- Device is not blocked.
- Current device/account policy is not exceeded.
- Nonce/request is valid.

### Local checks before execution

1. Verify lease signature.
2. Match Product ID to installed package.
3. Match installed version to `versionRange`.
4. Match device identity.
5. Check current trusted time against `offlineUntil`.
6. Check entitlement expiry embedded in the lease.
7. Match package version/hash to the installed verified state.

## 10. UAT lease policy

| Parameter | Baseline |
|---|---|
| `refreshAfter` | 24 hours |
| `offlineUntil` | 72 hours |

Behavior:

| Situation | Result |
|---|---|
| Online, lease valid | Run immediately; refresh in background after threshold |
| Offline, lease valid | Run if required capabilities can work offline |
| Offline, lease expired | Ask Customer to connect |
| Server down, lease valid | Continue to run |
| Server down, lease expired | Only a signed grace policy may allow it |
| Entitlement expired | Do not issue a new lease |
| Product revoked | Block when trusted revocation state is received, subject to signed offline policy |
| Clock moves backward | Require online trust refresh |
| User logs out | Remove or invalidate local lease and account-bound credential session |
| Long-running execution | Recheck periodically or before sensitive capability, according to policy |

## 11. Important distinction

Entitlement and Lease are not duplicates:

- Entitlement is the durable server business record.
- Lease is a signed, bounded proof usable by a device online or offline.

## 12. Open decisions

- Device identity strength and later Play Integrity usage.
- Number of devices per account/entitlement.
- Emergency revocation behavior while a device remains offline.
- Customer credential vault location and encryption design.
- Token refresh ownership: Launcher, platform service, or provider adapter.
- Whether Runtime receives token material or only a callable credential proxy.
- Long-running execution lease recheck interval.

