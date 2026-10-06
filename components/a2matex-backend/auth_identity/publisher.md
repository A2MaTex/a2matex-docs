# A2MaTex Backend — Publisher Module

## 1. Mục tiêu

Publisher Module quản lý khả năng xuất bản sản phẩm của một user.

- Mọi user `ACTIVE` đều có thể tự kích hoạt Publisher.
- User giữ role `CUSTOMER` và được gán thêm role `PUBLISHER`.
- Admin không duyệt tư cách Publisher; admin review từng Product Submission.
- Admin vẫn có thể block riêng quyền xuất bản mà không ban toàn bộ user.
- Thông tin công khai lấy từ `User + Profile`, không sao chép sang Publisher.

## 2. Quan hệ dữ liệu

```text
User
├── Profile
├── UserRole → PUBLISHER
└── PublisherAccount
    ├── PublisherTermsAcceptance[]
    └── Product[]
```

`PublisherAccount` là tài khoản nghiệp vụ phục vụ xuất bản, không phải publisher profile.

Thông tin hiển thị:

| Dữ liệu              | Nguồn                    |
| -------------------- | ------------------------ |
| Username             | `User.username`          |
| Tên hiển thị         | `Profile.fullName`       |
| Avatar               | `Profile.avatarStaticId` |
| Giới thiệu           | `Profile.bio`            |
| Website/social links | `Profile`                |
| Email liên hệ        | `Profile.email`          |

Không công khai `User.email`, `Profile.phone`, `Profile.address` hoặc `Profile.birthday`.

## 3. Database schema

### 3.1. PublisherAccountStatus

```sql
CREATE TYPE "PublisherAccountStatus" AS ENUM (
    'ACTIVE',
    'BLOCKED'
);
```

### 3.2. PublisherAccount

```sql
CREATE TABLE "PublisherAccount" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId" UUID NOT NULL UNIQUE REFERENCES "User" (id),
    "status" "PublisherAccountStatus" NOT NULL DEFAULT 'ACTIVE',
    "activatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    "blockedReason" TEXT,
    "blockedAt" TIMESTAMPTZ,
    "blockedById" UUID REFERENCES "User" (id),

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ
);
```

Quy tắc:

- Một user có tối đa một `PublisherAccount`.
- Không tạo lại record khi bị block.
- `BLOCKED` chỉ được chuyển lại `ACTIVE` bởi admin.
- Product sau này tham chiếu `Product.publisherAccountId → PublisherAccount.id`.

### 3.3. Term

```sql
CREATE TYPE "TermType" AS ENUM (
    'PUBLISHER_TERMS',
    'CUSTOMER_TERMS',
    'PRIVACY_POLICY'
);

CREATE TYPE "TermStatus" AS ENUM (
    'DRAFT',
    'ACTIVE',
    'ARCHIVED'
);

CREATE TABLE "Term" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    type "TermType" NOT NULL,
    version VARCHAR(100) NOT NULL,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,

    status "TermStatus" NOT NULL DEFAULT 'DRAFT',
    "publishedAt" TIMESTAMPTZ,

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ,
    "deletedAt" TIMESTAMPTZ,

    "createdById" UUID NOT NULL REFERENCES "User" (id),
    "updatedById" UUID REFERENCES "User" (id),
    "deletedById" UUID REFERENCES "User" (id),

    CONSTRAINT "Term_type_version_key"
        UNIQUE (type, version)
);
```

Quy tắc:

- Mỗi `TermType` chỉ có một bản `ACTIVE` tại một thời điểm.
- Term `DRAFT` được chỉnh sửa.
- Term `ACTIVE` và `ARCHIVED` là immutable.
- Khi publish bản mới, bản `ACTIVE` cũ cùng type chuyển thành `ARCHIVED` trong cùng transaction.

### 3.4. PublisherTermsAcceptance

```sql
CREATE TABLE "PublisherTermsAcceptance" (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    "publisherAccountId" UUID NOT NULL
        REFERENCES "PublisherAccount" (id),

    "termId" UUID NOT NULL
        REFERENCES "Term" (id),

    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "PublisherTermsAcceptance_account_term_key"
        UNIQUE ("publisherAccountId", "termId")
);
```

## 4. Điều kiện sử dụng quyền Publisher

Một user được tạo Product hoặc gửi Submission khi thỏa mãn toàn bộ:

```text
User.status = ACTIVE
AND có UserRole PUBLISHER
AND Role PUBLISHER đang ACTIVE
AND PublisherAccount.status = ACTIVE
AND đã chấp nhận Term ACTIVE loại PUBLISHER_TERMS
```

Nên đóng gói kiểm tra trên thành một application policy dùng chung:

```ts
publisherAccessPolicy.assertActivePublisher(userId);
```

Product và Submission Module phải gọi policy này, không chỉ kiểm tra role.

## 5. API contract

### 5.1. Lấy trạng thái Publisher hiện tại

```http
GET /publishers/me
Authorization: Bearer <access-token>
```

Khi chưa kích hoạt:

```json
{
  "publisherAccount": null,
  "missingFields": ["profile.fullName", "profile.email"],
  "currentTerms": {
    "id": "term_01",
    "version": "2026-01",
    "title": "A2MaTex Publisher Terms"
  }
}
```

Khi đã kích hoạt:

```json
{
  "publisherAccount": {
    "id": "publisher_01",
    "status": "ACTIVE",
    "activatedAt": "2026-09-29T01:00:00Z"
  },
  "acceptedCurrentTerms": true,
  "currentTerms": {
    "id": "term_01",
    "version": "2026-01",
    "title": "A2MaTex Publisher Terms"
  },
  "profile": {
    "username": "dino",
    "fullName": "Dino",
    "email": "contact@example.com",
    "avatarStaticId": null,
    "bio": null,
    "website": null
  }
}
```

### 5.2. Kích hoạt Publisher

```http
POST /publishers/activate
Authorization: Bearer <access-token>
```

Request:

```json
{
  "termId": "term_01",
  "acceptedTerms": true
}
```

Transaction:

```text
1. Lock User.
2. Kiểm tra User ACTIVE.
3. Kiểm tra Profile.fullName và Profile.email.
4. Kiểm tra acceptedTerms = true.
5. Kiểm tra Term là bản ACTIVE loại PUBLISHER_TERMS.
6. Tạo PublisherAccount ACTIVE nếu chưa tồn tại.
7. Nếu account đã BLOCKED thì từ chối.
8. Tạo PublisherTermsAcceptance nếu chưa tồn tại.
9. Đảm bảo user có role PUBLISHER đang ACTIVE.
10. Commit.
```

API phải idempotent:

- Đã `ACTIVE` và đã chấp nhận terms: trả account hiện tại.
- Role đã tồn tại: không tạo `UserRole` trùng.
- Terms acceptance đã tồn tại: không tạo record trùng.

User bổ sung thông tin thiếu qua API đã có:

```http
PATCH /users/me/profile
```

Không tạo API cập nhật publisher profile riêng.

### 5.3. Chấp nhận điều khoản mới

```http
POST /publishers/terms/accept
Authorization: Bearer <access-token>
```

Request:

```json
{
  "termId": "term_02",
  "acceptedTerms": true
}
```

Quy tắc:

- PublisherAccount phải tồn tại và không bị `BLOCKED`.
- Chỉ chấp nhận Term `ACTIVE` loại `PUBLISHER_TERMS`.
- API idempotent theo `publisherAccountId + termId`.
- Publisher chưa chấp nhận terms mới được xem portal nhưng không được tạo Product hoặc Submission mới.

### 5.4. Lấy điều khoản hiện tại

```http
GET /terms/current?type=PUBLISHER_TERMS
```

Response:

```json
{
  "id": "term_01",
  "type": "PUBLISHER_TERMS",
  "version": "2026-01",
  "title": "A2MaTex Publisher Terms",
  "content": "Markdown content...",
  "publishedAt": "2026-09-29T01:00:00Z"
}
```

### 5.5. Public Publisher profile

```http
GET /marketplace/publishers/{username}
```

Backend đọc:

```text
User theo username
→ PublisherAccount
→ Profile
→ public Product summary
```

Chỉ trả các field Profile nằm trong public whitelist. Việc có hiển thị Publisher `BLOCKED` hay không phụ thuộc vào việc họ còn Product công khai, do Product Module quyết định.

### 5.6. Admin lấy danh sách

```http
GET /publishers?status=ACTIVE&page=1&pageSize=10
```

Trả dữ liệu tổng hợp từ:

```text
PublisherAccount + User + Profile
```

### 5.7. Admin xem chi tiết

```http
GET /publishers/{publisherAccountId}
```

Trả account status, user status, profile, role và terms version đã chấp nhận.

### 5.8. Admin cập nhật trạng thái

```http
PATCH /publishers/{publisherAccountId}/status
```

Block:

```json
{
  "status": "BLOCKED",
  "reason": "Repeated policy violations"
}
```

Unblock:

```json
{
  "status": "ACTIVE"
}
```

Transaction:

```text
1. Lock PublisherAccount.
2. Kiểm tra current status != status gửi tới.
3. Validate transition.
4. BLOCKED yêu cầu phải có reason.
5. Cập nhật status và block metadata.
6. Khi ACTIVE, đảm bảo role PUBLISHER tồn tại và active.
7. Ghi audit event.
8. Commit.
```

Không xóa role khi block để user vẫn xem được portal và lý do bị block. Mọi mutation Product/Submission bị chặn bởi `publisherAccessPolicy`.

### 5.9. Admin quản lý Terms

Tạo bản draft:

```http
POST /terms
```

```json
{
  "type": "PUBLISHER_TERMS",
  "version": "2026-02",
  "title": "A2MaTex Publisher Terms",
  "content": "Markdown content..."
}
```

Sửa nội dung draft:

```http
PUT /terms/{termId}
```

Chỉ Term `DRAFT` được sửa.

Publish hoặc archive bằng một API status:

```http
PUT /terms/{termId}/status
```

```json
{
  "status": "ACTIVE"
}
```

Khi publish trong một transaction:

```text
1. Lock Term được publish.
2. Kiểm tra Term đang DRAFT.
3. Chuyển bản ACTIVE cũ cùng type thành ARCHIVED.
4. Chuyển Term mới thành ACTIVE và đặt publishedAt.
5. Commit.
```

## 6. Allowed status transitions

```text
ACTIVE  → BLOCKED  // admin
BLOCKED → ACTIVE   // admin
```

User không có API tự block hoặc unblock PublisherAccount.

## 7. Edge cases

| Trường hợp                                   | Xử lý                                          |
| -------------------------------------------- | ---------------------------------------------- |
| User `INACTIVE` hoặc `BANNED`                | Không được activate hoặc publish               |
| Gọi activate nhiều lần                       | Trả account hiện tại                           |
| Publisher bị block gọi activate              | Từ chối, không tạo account mới                 |
| Role PUBLISHER đã có                         | Không tạo mapping trùng                        |
| Role PUBLISHER bị inactive toàn hệ thống     | Không activate/publish                         |
| Term client gửi không còn ACTIVE             | Yêu cầu tải Term hiện tại                      |
| Terms được cập nhật sau khi Publisher active | Chặn Product/Submission mới cho đến khi accept |
| Hai admin đổi status đồng thời               | Request có `expectedStatus` sai trả conflict   |
| Admin block Publisher                        | Không tự động ban User                         |
| User thay đổi Profile                        | Public publisher profile dùng dữ liệu mới      |

## 8. Test bắt buộc

- [ ] User ACTIVE với Profile đầy đủ activate thành công.
- [ ] User INACTIVE/BANNED không activate được.
- [ ] Thiếu `Profile.fullName` hoặc `Profile.email` bị từ chối.
- [ ] Term không còn ACTIVE hoặc chưa accept bị từ chối.
- [ ] Activate tạo account, acceptance và role trong cùng transaction.
- [ ] Activate lặp lại không tạo dữ liệu trùng.
- [ ] Publisher BLOCKED không tự activate lại.
- [ ] Accept terms mới thành công và idempotent.
- [ ] Chưa accept terms hiện tại không tạo Product/Submission được.
- [ ] Admin block bắt buộc reason.
- [ ] Admin unblock đảm bảo role PUBLISHER tồn tại.
- [ ] status không khớp trả conflict.
- [ ] Public profile không lộ thông tin cá nhân private.
- [ ] Block Publisher không ảnh hưởng quyền Customer.
- [ ] Chỉ Term DRAFT được chỉnh sửa.
- [ ] Publish Term mới archive bản ACTIVE cũ trong cùng transaction.

## 11. Quyết định đã chốt

1. Publisher profile không có bảng riêng; dùng `User + Profile`.
2. `PublisherAccount` chỉ quản lý quyền xuất bản và trạng thái block.
3. Mọi user ACTIVE có thể tự kích hoạt Publisher, không cần admin approve.
4. Publisher giữ role CUSTOMER và được gán thêm PUBLISHER.
5. Admin review Product Submission, không review tư cách Publisher.
6. Admin có thể block riêng quyền Publisher mà không ban User.
7. Product sau này tham chiếu `PublisherAccount.id`.
8. Không bổ sung index hiệu năng trong giai đoạn thiết kế này.
