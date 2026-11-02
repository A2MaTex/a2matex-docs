# A2MaTex MVP — System Checklist

Tài liệu này là bản đồ tổng quan để chia phạm vi MVP theo platform. A2MaTex hiện gồm năm platform:

| Platform                     | Trách nhiệm chính                                                                               |
| ---------------------------- | ----------------------------------------------------------------------------------------------- |
| **Web Marketplace Frontend** | Hiển thị dữ liệu từ Backend API cho guest, customer, publisher và admin                         |
| **Backend**                  | Cung cấp toàn bộ API, business rule, authorization, persistence và integration cho các platform |
| **Studio**                   | Thiết kế, validate, test và lưu project automation lên Backend                                  |
| **Launcher**                 | Quản lý product đã sở hữu, package, cấu hình chạy và giao tiếp với Runtime Agent                |
| **Runtime Agent**            | Phát hiện device và thực thi Runtime IR trên máy customer                                       |

Các module Backend nằm trong cùng NestJS modular monolith, chưa tách thành microservice. Chi tiết database, state machine và workflow nghiệp vụ sẽ được thiết kế sau khi hoàn tất plan/checklist tổng thể.

Mục tiêu MVP đầu tiên là hoàn thành hai hành trình chính:

1. Customer kích hoạt Publisher → phát triển project trong Studio → lưu project → tự review và tạo submission trên Web → admin review → publish Product.
2. Customer nhận/mua Product trên Marketplace → mở Launcher → tải Product → chọn device → Runtime Agent chạy workflow trên máy cá nhân.

---

# 1. Backend — API và business modules

Backend là source of truth cho dữ liệu cloud và cung cấp API cho Web Marketplace, Studio, Launcher và Runtime Agent.

## 1.1. Identity, account và access

- [x] **Auth Module** — login, logout, access/refresh token và session.
- [x] **User/Profile Module** — thông tin customer, profile công khai và trạng thái tài khoản.
- [x] **RBAC Module** — role và permission; resource ownership vẫn được kiểm tra trong domain service tương ứng.
- [x] **Publisher Module** — `PublisherAccount`, self-activation, block/unblock và shared publisher access policy.
- [x] **Terms Module** — versioned terms, publish/archive và immutable terms acceptance.
- [x] **Launcher Release Module** — quản lý version, Static package và active Launcher release để public download.

## 1.2. Studio project và workflow definition

- [x] **Node Catalog Module** — system-owned current node definitions, config/input/output schema, capability và Runtime compatibility. Xem [`studio_project_workflow/node_catalog.md`](studio_project_workflow/node_catalog.md).
- [x] **Project Module** — Publisher-owned cloud project container, metadata, ownership và archive lifecycle đã triển khai, migrate, gán permission và smoke-test CRUD; các case nhiều user còn được theo dõi trong spec. Xem [`studio_project_workflow/project_module.md`](studio_project_workflow/project_module.md).
- [x] **Project Draft Module** — mutable workflow draft do Studio lưu/sync lên Backend; migration,
      permission, optimistic concurrency và compiled HTTP behavior đã được verify. Studio conflict-tab UX
      được để lại cho client integration. Xem
      [`studio_project_workflow/project_draft.md`](studio_project_workflow/project_draft.md).
- [x] **Workflow Revision Module** — private service đã implement migration, exact-version locking,
      immutable snapshot, deterministic hash và focused tests; atomic integration với Submission
      được để lại cho Submission Module. Xem
      [`studio_project_workflow/workflow_revision.md`](studio_project_workflow/workflow_revision.md).
- [ ] **Asset Module** — metadata và ownership của template image, OCR resource, icon, screenshot và video.
- [ ] **Storage Service** — upload lifecycle, Object Storage object key và presigned URL.
- [x] **Basic Workflow Validation Service** — engine nội dung độc lập cho Editor Graph schema v1,
      topology, active node, source port và static config; test API và submission worker dùng chung
      service contract, với diagnostics có giới hạn và deterministic. Xem
      [`studio_project_workflow/workflow_validation.md`](studio_project_workflow/workflow_validation.md).
- [ ] **Advanced Workflow Validation** — kiểm tra binding, conditional/loop/error semantics và
      Runtime IR consistency trong các phase tiếp theo.

Studio chỉ lưu và đồng bộ project. Studio không gửi Product Submission trực tiếp.

## 1.3. Product, submission và review

- [x] **Product Module** — Publisher-owned editable aggregate: identity, Project link, name, slug,
      descriptions, usage instructions, shared Static-backed ordered image media và lifecycle đã
      implement; manual verification/live permission check pending, automated tests tạm hoãn.
- [ ] **Submission Module** — nhận project được chọn trên Web, tạo immutable Workflow Revision và tạo Submission tham chiếu revision đó.
- [ ] **Automated Scanner Service** — capability, graph/IR, node catalog contract/hash, code node, asset và policy scan.
- [ ] **Review Module** — scan result, admin feedback, request changes, approve và reject.
- [ ] **Release Module** — version và immutable `ProductRelease`.
- [ ] **Package Builder Service** — chuẩn hóa Runtime IR, tạo manifest và đóng gói `.a2m`.
- [ ] **Encryption/Key Service** — mã hóa workflow payload và quản lý release key.
- [ ] **Artifact Integrity Service** — tạo package hash, payload hash và metadata để Launcher/Runtime verify.
- [ ] **Catalog/Search Module** — `@IsPublic()` list/detail cho Product `ACTIVE`, đọc Product snapshot
      của current published release, publisher page và tìm kiếm text cơ bản.

Category, Tag, ProductTag, required-app classification và connector taxonomy để sau MVP. Node hiện
tại chỉ có Android action cơ bản; chưa có taxonomy ổn định đáng để thêm table/API quản trị.

Luồng tạo submission trên Web:

```text
Publisher mở Publisher Portal
→ chọn project đã được Studio lưu trên Backend
→ tạo/chọn Product draft của project
→ lưu Product text và ordered image media bằng Static IDs
→ xem project summary và local validation/test summary mang tính tham khảo
→ tự review project theo checklist
→ nhập Release metadata
→ xác nhận submit
→ Backend lock existing Product row
→ Backend tạo immutable Workflow Revision từ đúng draft version
→ Backend tạo Submission snapshot Product/media, revision và release request
→ Backend đẩy background job validate Workflow Revision
→ lưu cloud validation result; chỉ tiếp tục scan/build khi pipeline cho phép
```

Nếu admin yêu cầu thay đổi, publisher sửa project trong Studio, lưu draft mới rồi tạo submission mới trên Web. Submission và revision cũ không bị ghi đè.

## 1.4. Distribution, entitlement và execution authorization

- [ ] **Product Offer Module** — Publisher replace toàn bộ ONE_TIME/SUBSCRIPTION collection; Backend
      update current price in-place, insert natural key mới, soft-delete offer bị omit và enforce
      Product-level cooldown từ validated environment config. Deleted Offer là immutable và không
      restore. Mutation chỉ trả `SUCCESS_RESPONSE`; management/public Product detail preload Offer
      hiện tại, còn public Product list chỉ preload minimum SUBSCRIPTION price và duration.
- [ ] **Order Module** — mua bằng `Profile.balance`, kể cả zero-value Order; snapshot Offer, block
      purchase khi Product entitlement còn active và enforce ONE_TIME một lần/customer/Product.
- [ ] **Entitlement Module** — quyền dài hạn của customer đối với Product.
- [ ] **Manual Grant Use Case** — admin cấp entitlement cho UAT/MVP; nằm trong Entitlement Module.
- [ ] **Download Authorization Service** — kiểm tra entitlement và cấp presigned package download URL.
- [ ] **Launcher API** — My Products, Product release metadata và compatibility.
- [ ] **Execution Lease Module** — issue, validate, renew, expire và revoke lease.
- [ ] **Lease Signing Service** — ký lease bound với customer, Product Release, package hash và Runtime session; hỗ trợ signing-key rotation.
- [ ] **Runtime Key Envelope Service** — wrap release content key bằng ephemeral Runtime public key; không trả plaintext key cho Launcher.
- [ ] **Production Authorization API** — issue/renew signed lease và key envelope theo [Production Execution Authorization v1](../../specifications/production_execution_authorization_v1.md).
- [ ] **Execution Report Module** — nhận idempotent run summary từ Runtime Agent.

## 1.5. Commerce — phạm vi MVP tối thiểu

- [ ] **Zero-value ONE_TIME Acquisition** — đi cùng Order path, không cấp Entitlement trực tiếp và
      không tạo PaymentTransaction.
- [ ] **Order history** — giữ immutable Offer snapshot; không mở rộng promotion/refund trong MVP.
- [ ] **Payment Module** — để sau UAT nếu payment gateway production chưa cần trong tháng đầu.

MVP ưu tiên free Product và manual entitlement grant. Không để payment production chặn hành trình publish/download/run.

## 1.6. Moderation và supporting capabilities — giữ đơn giản

- [ ] **Product Report/Moderation Module** — customer gửi report, admin xem và resolve với reason.
- [ ] **Audit Log Module** — ghi các mutation nhạy cảm: publisher status, review, release, entitlement và moderation.
- [ ] **Job Queue** — scan, package build, cleanup và retry.
- [ ] **Transactional Outbox** — chỉ áp dụng cho transaction cần phát job/event không được phép mất.
- [ ] **Idempotency** — áp dụng cho activate publisher, save/finalize revision, submission, grant, lease và execution report khi cần.
- [ ] **Notification Module** — notification trong ứng dụng cho review result và release; email/push nâng cao có thể để sau MVP.
- [ ] **Cache** — chỉ cache catalog, node definition hoặc public data khi có nhu cầu thực tế.
- [ ] **Observability** — structured log, request correlation và lỗi job tối thiểu cho MVP.
- [ ] **Configuration/Secret** — Object Storage key, encryption key và external service secret.

---

# 2. Web Marketplace Frontend

Web Marketplace Frontend không sở hữu business rule. Frontend hiển thị Backend API response, thu thập user input và thể hiện trạng thái nghiệp vụ.

## 2.1. Public Marketplace

- [ ] Product catalog và text search cơ bản.
- [ ] Product catalog preload minimum SUBSCRIPTION price và duration, không dùng ONE_TIME cho list.
- [ ] Product detail preload toàn bộ current Offer và compatibility; không có Offer GET API riêng.
- [ ] Public publisher profile.
- [ ] Download active Launcher release.
- [ ] Login/register entry points.

## 2.2. Customer Portal

- [ ] My Products và entitlement status.
- [ ] Product acquisition/purchase entry.
- [ ] Order/ownership history cơ bản.
- [ ] Open Product in Launcher.
- [ ] Submit Product report.

## 2.3. Publisher Portal

- [ ] Publisher activation và current terms acceptance.
- [ ] Publisher status/blocked reason.
- [ ] Danh sách project đã được Studio lưu lên Backend.
- [ ] Project summary và self-review checklist.
- [ ] Tạo/chọn Product; nhập Product text và chọn ordered image media đã upload qua Static/R2.
- [ ] Nhập release version, changelog và compatibility.
- [ ] Review toàn bộ submission data trước khi gửi.
- [ ] Submission history, scan result và admin feedback.
- [ ] Product draft và release history.

## 2.4. Admin Portal

- [ ] Publisher account và Terms management.
- [ ] Submission queue và review detail.
- [ ] Scan findings, feedback, request changes, approve và reject.
- [ ] Product/release suspend hoặc disable.
- [ ] Launcher Release list, detail, create, update và activate.
- [ ] Manual entitlement grant/revoke.
- [ ] Product report resolution.
- [ ] Audit log view cơ bản.

Chi tiết màn hình nằm trong `components/ui_checklist.md`.

---

# 3. Studio

Studio chịu trách nhiệm tạo project automation và lưu project lên Backend, không trực tiếp tạo Submission.

## 3.1. Project authoring

- [ ] Create/open local project.
- [ ] Project explorer, Graph Canvas và Node Palette.
- [ ] Schema-driven Node Inspector.
- [ ] Workflow input, variable, binding và asset management.
- [ ] Controlled branch, error edge và loop.
- [ ] Local autosave và crash recovery.

## 3.2. Validation và testing

- [ ] Load compatible Node Catalog từ Backend/cache.
- [ ] Validate Editor Graph trong lúc thiết kế.
- [ ] Compile Editor Graph thành versioned Runtime IR.
- [ ] Runtime Agent development-mode connection.
- [ ] Single-node, run-from-node và full-workflow test.
- [ ] Execution inspector, output, artifact và diagnostics.

## 3.3. Cloud synchronization và Web handoff

- [ ] Create/link cloud Project.
- [ ] Save/sync mutable project draft lên Backend.
- [ ] Upload assets và lưu asset references.
- [ ] Hiển thị save/sync state và conflict.
- [ ] Mở project tương ứng trong Publisher Portal để self-review và tạo Submission.
- [ ] Không có luồng submit thẳng từ Studio trong MVP.

---

# 4. Launcher

- [ ] Login/logout/refresh session.
- [ ] My Products từ Backend.
- [ ] Download package bằng presigned URL.
- [ ] Verify package hash, manifest và compatibility.
- [ ] Installed release management và atomic update.
- [ ] Runtime Agent supervisor và handshake.
- [ ] Device list từ Runtime Agent.
- [ ] Input profile và credential reference.
- [ ] Per-device run configuration.
- [ ] Request/renew execution lease.
- [ ] Start/reconnect/pause/resume/cancel run qua Runtime Agent.
- [ ] Execution monitor và `PAUSED`/`AUTHORIZATION_EXPIRED` reason.
- [ ] Run summary visibility.

Launcher không gọi ADB trực tiếp và không giữ execution state làm source of truth.

---

# 5. Runtime Agent

- [ ] Local authenticated HTTP/event API cho Studio và Launcher.
- [ ] LDPlayer và bundled ADB discovery.
- [ ] Device Registry và per-device exclusive execution lock.
- [ ] Runtime IR validator và Handler Registry.
- [ ] Single-device workflow engine.
- [ ] Multi-device scheduler và run-group coordination.
- [ ] Typed Android commands; không nhận raw shell/ADB command.
- [ ] Flow, Android action, observation và connector handlers tối thiểu.
- [ ] Timeout, retry, pause, cancel và controlled loop.
- [ ] SQLite checkpoint, node result, event và artifact metadata.
- [ ] WebSocket/SSE event stream cho Studio/Launcher.
- [ ] Production package verification, lease validation và in-memory decryption.
- [ ] Idempotent execution summary sync về Backend.

Runtime Agent quản lý LDPlayer/ADB device và execution detail trên máy customer. Backend quản lý Launcher release, lease và execution summary cloud nhưng không đăng ký Launcher installation.

---

# 6. Hai hành trình MVP cấp cao

Phần này chỉ giữ overview. Workflow chi tiết, error branch và state transition sẽ được thiết kế sau.

## 6.1. Customer trở thành Publisher và publish Product

```text
Customer
→ activate Publisher và accept Terms trên Web
→ tạo/phát triển project trong Studio
→ Studio save/sync project và assets lên Backend
→ Publisher chọn project trên Web
→ tự review project và local validation/test summary
→ nhập Product draft và Release data
→ submit
→ Backend tạo immutable Workflow Revision và Submission
→ background cloud validation
→ automated scan
→ admin review
→ build/package/encrypt ProductRelease
→ publish Product trên Marketplace
```

## 6.2. Customer sở hữu và chạy Product

```text
Customer xem/nhận/mua Product trên Web
→ Backend tạo balance/zero-value Order và snapshot Product Offer
→ Backend cấp hoặc reactivate một durable Entitlement
→ customer mở Launcher
→ Launcher tải Product package
→ Launcher verify package và chọn Runtime devices
→ Launcher request execution lease
→ Runtime validate lease và decrypt Runtime IR trong memory
→ Runtime chạy workflow trên từng device
→ Launcher hiển thị execution state
→ Runtime gửi run summary về Backend
```

---

# 7. Cross-platform contracts cần chốt trước implementation

- [ ] Editor Graph schema và schema version.
- [x] Node Definition, unique node `type`, capability vocabulary và Runtime compatibility cho MVP.
- [ ] Runtime IR schema và compiler contract.
- [ ] Project draft version/conflict contract.
- [ ] Workflow Revision immutable snapshot contract.
- [ ] Submission payload và self-review checklist contract.
- [ ] Product snapshot/Release metadata contract.
- [ ] `.a2m` manifest, payload hash và package hash contract.
- [ ] Minimum Studio/Launcher/Runtime compatibility rules.
- [ ] Launcher ↔ Runtime local API contract.
- [ ] Execution lease và Runtime validation contract.
- [ ] Execution event và run summary contract.
- [ ] API error format, pagination và generic success response conventions.

Database ownership, entity relation, constraint, indexes, retention và detailed state machine sẽ được thiết kế sau khi plan/checklist tổng thể được duyệt.

---

# 8. Thứ tự thực hiện đề xuất cho MVP một tháng

1. Chốt cross-platform JSON/API contracts cần cho vertical slice.
2. Publisher Account + Terms + Web activation UI.
3. Node Catalog contract tối thiểu. **Đã hoàn thành ở Backend.**
4. Project cloud container + ownership + archive lifecycle.
5. Project Draft + Asset upload + Studio save/sync.
6. Product aggregate/media tối thiểu + Web project selection/self-review.
7. Workflow Revision + Submission + scan/validation.
8. Admin Review + request changes/approve/reject.
9. ProductRelease + package/encryption + publish.
10. Product Offer + balance Order + Entitlement + My Products.
11. Launcher Release management + Launcher/Product package download/verification.
12. Execution Lease + Launcher/Runtime integration.
13. Runtime execution + event monitor + execution summary.
14. Minimal moderation, audit, notification và operational hardening.

Payment production, advanced search, recommendations, rating/review, coupon, payout, publisher team, arbitrary code node và remote execution control không thuộc critical path của MVP đầu tiên.
