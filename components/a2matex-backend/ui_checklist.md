# A2MaTex MVP — UI Checklist

Tài liệu này liệt kê UI surface cần thiết cho MVP một tháng. Business rule và authorization nằm trong Backend; UI chỉ gọi API, thu thập input, hiển thị response và hướng dẫn user qua đúng trạng thái nghiệp vụ.

Năm platform được tính trong plan:

1. Web Marketplace Frontend.
2. Backend — không có product UI riêng, cung cấp API cho các UI khác.
3. Studio Desktop.
4. Launcher Desktop.
5. Runtime Agent — không có end-user UI độc lập trong MVP; state được hiển thị qua Studio/Launcher.

---

# 1. Shared UI rules

- [ ] Dùng cùng terminology cho Product, Project, Draft, Revision, Submission, Product Release, Launcher Release, Entitlement, Device và Execution.
- [ ] Không hiển thị raw internal enum/error trực tiếp cho user.
- [ ] Mọi mutation có disabled/loading state và chống submit lặp.
- [ ] Form hiển thị field-level validation từ client và Backend.
- [ ] Có empty, loading, success, error, forbidden và not-found state.
- [ ] Có conflict/stale-version state cho project draft và status update.
- [ ] Blocked, suspended, revoked, expired và incompatible phải có reason/action rõ ràng.
- [ ] Không đưa access token, refresh token, lease token, workflow key hoặc credential vào URL/log/UI state không cần thiết.
- [ ] Destructive/sensitive action cần confirmation và reason khi business rule yêu cầu.
- [ ] MVP ưu tiên desktop layout cho Studio/Launcher và responsive layout cơ bản cho Web Marketplace.

---

# 2. Web Marketplace Frontend

## 2.1. Public screens — P0

- [ ] Home/Product catalog chỉ hiển thị Product `ACTIVE` có current published release hợp lệ.
- [ ] Text search cơ bản.
- [ ] Product catalog hiển thị minimum SUBSCRIPTION price và duration; không dùng ONE_TIME Offer cho
      list projection.
- [ ] Product detail: description, publisher, current ONE_TIME/SUBSCRIPTION offers, compatibility và
      current release; Product không có offer vẫn visible nhưng không purchasable.
- [ ] Public publisher profile.
- [ ] Download active Launcher release và hiển thị version hiện tại.
- [ ] Login/register entry.

## 2.2. Customer Portal — P0

- [ ] My Products.
- [ ] Entitlement status và expiration; MVP không hiển thị machine/device/concurrency limit.
- [ ] Zero-value hoặc balance purchase entry; disable purchase khi customer còn active Entitlement.
- [ ] Open in Launcher action.
- [ ] Order/ownership history cơ bản.
- [ ] Account/profile settings dùng chung.

## 2.3. Publisher activation — P0

- [ ] Publisher introduction/eligibility page.
- [ ] Missing Profile fields state và link cập nhật Profile.
- [ ] Current Publisher Terms viewer.
- [ ] Accept terms + activate Publisher confirmation.
- [ ] Publisher status page.
- [ ] Blocked state và blocked reason.
- [ ] New terms acceptance required state.

## 2.4. Publisher Portal — P0

- [ ] Publisher dashboard tối giản.
- [ ] Cloud Project list do Studio lưu lên Backend.
- [ ] Project detail summary: saved version, graph summary, assets, validation, capabilities và test summary.
- [ ] Submission review-context form.
- [ ] Submission wizard:
  - [ ] Step 1 — chọn Project và exact draft version.
  - [ ] Step 2 — xem validation/capability/asset summary.
  - [ ] Step 3 — tạo/chọn Product draft; nhập Product text và ordered image media bằng Static IDs.
  - [ ] Step 4 — nhập release version, changelog và minimum compatibility.
  - [ ] Step 5 — review Product snapshot, media, Workflow Revision và confirm submit.
- [ ] Submission history.
- [ ] Submission detail: immutable revision, scan findings, status và admin feedback.
- [ ] Request-changes handoff: mở đúng Project trong Studio để sửa.
- [ ] My Products.
- [ ] Product draft editor tối thiểu; save theo MVP last-write-wins và không thay đổi release đang public.
- [ ] Product Offer collection editor: replace complete type/price/duration set, handle Backend
      cooldown rejection, không hiển thị deleted Offer trong default dashboard, và reload Product
      detail sau `SUCCESS_RESPONSE` để lấy state mới nhất.
- [ ] Release history và current release.

## 2.5. Admin Portal — P0

- [ ] Submission review queue.
- [ ] Submission review detail:
  - [ ] Product snapshot gồm presentation fields và ordered media.
  - [ ] Project/revision summary.
  - [ ] Required capabilities.
  - [ ] Scan findings grouped by error/warning/info.
  - [ ] Assets và review instructions.
  - [ ] Feedback/reason input.
  - [ ] Request changes, reject và approve actions.
- [ ] Publisher Account list/detail và block/unblock.
- [ ] Terms list, draft editor và publish action.
- [ ] Product/release list, current Offer read-only view và suspend/disable action.
- [ ] Launcher Release list/detail, create/update và activate action.
- [ ] Manual Entitlement grant/revoke.

## 2.6. Supporting Web screens — P1 if time allows

- [ ] Product Report submission.
- [ ] Admin Product Report list/detail/resolve.
- [ ] In-app notification list.
- [ ] Audit log viewer cơ bản.
- [ ] Featured product management.

---

# 3. Studio Desktop

## 3.1. App and project shell — P0

- [ ] Authentication/session state nếu Studio yêu cầu cloud API.
- [ ] Local Project list/create/open.
- [ ] Cloud Project link/create state.
- [ ] Project Explorer.
- [ ] Save locally và save/sync to Backend.
- [ ] Save status: unsaved, saving, synced, failed và conflict.

## 3.2. Workflow editor — P0

- [ ] Graph Canvas: add, connect, select, move, delete, zoom và pan.
- [ ] Node Palette: search, category, version và capability indicator.
- [ ] Schema-driven Node Inspector.
- [ ] Input mapping/expression picker.
- [ ] Retry, timeout và error policy controls.
- [ ] Workflow input and variable editor.
- [ ] Asset browser/upload/reference.
- [ ] Inline graph diagnostics.

## 3.3. Test and diagnostics — P0

- [ ] Runtime Agent connection/compatibility state.
- [ ] Device selector.
- [ ] Run selected node.
- [ ] Run from node.
- [ ] Run full workflow.
- [ ] Current node/branch/loop visualization.
- [ ] Node input/output/error inspector.
- [ ] Screenshot/artifact viewer.
- [ ] Stop/cancel test execution.

## 3.4. Web submission handoff — P0

- [ ] Validate before handoff.
- [ ] Show blocking errors and warnings.
- [ ] Save latest project draft to Backend.
- [ ] Open the selected Project in Publisher Portal.
- [ ] Clearly state that Product data and final Submission are completed on Web.
- [ ] Studio must not directly create Product Submission in MVP.

## 3.5. Studio P1 if time allows

- [ ] Undo/redo history UI.
- [ ] Auto-layout.
- [ ] Multiple test profiles.
- [ ] Multi-device smoke test.
- [ ] Advanced execution timeline.

---

# 4. Launcher Desktop

## 4.1. Setup and account — P0

- [ ] Login/logout/session-expired flow.
- [ ] Runtime Agent missing/starting/connected/incompatible/crashed states.
- [ ] LDPlayer installation path setup when discovery fails.

## 4.2. Product Library — P0

- [ ] My Products list.
- [ ] Product detail and usage instructions.
- [ ] Not installed/downloading/verifying/installed/update available/corrupted states.
- [ ] Install, retry download and update actions.
- [ ] Entitlement active/expired/revoked state.
- [ ] Launcher/Runtime incompatibility state.

## 4.3. Device and run configuration — P0

- [ ] Device list from Runtime Agent.
- [ ] Online, booting, offline, unauthorized and busy states.
- [ ] Select eligible devices.
- [ ] Dynamic input form generated from release input schema.
- [ ] Per-device input/profile assignment.
- [ ] Credential reference selection without exposing raw secrets.
- [ ] Preflight summary.
- [ ] License/concurrency validation error.
- [ ] Start Run action.

## 4.4. Execution monitor — P0

- [ ] Run-group summary: queued/running/succeeded/failed.
- [ ] Per-device execution status.
- [ ] Current node, sequence and loop iteration.
- [ ] Node error and artifact display.
- [ ] Pause/resume/cancel actions.
- [ ] Runtime reconnect and execution state reload.
- [ ] `PAUSED` với reason `AUTHORIZATION_EXPIRED` và renew/retry guidance.
- [ ] Final run summary.

## 4.5. Launcher P1 if time allows

- [ ] Saved input profiles.
- [ ] Device aliases.
- [ ] Download history.
- [ ] System tray/background behavior.
- [ ] Detailed historical execution viewer.

---

# 5. Runtime Agent UI boundary

Runtime Agent does not need a standalone customer-facing UI for the first MVP.

- [ ] Expose health, version, supported handlers and compatibility to Studio/Launcher.
- [ ] Expose device state and refresh progress.
- [ ] Expose run group, device execution, node execution and artifact events.
- [ ] Return stable error codes plus safe user-facing messages.
- [ ] Keep raw diagnostic detail available for logs without leaking credential, token or decrypted workflow data.
- [ ] If a local diagnostics page is added, keep it developer-only and outside the critical MVP path.

---

# 6. Cross-platform handoff checklist

- [ ] Studio can open the exact cloud Project in Publisher Portal.
- [ ] Web can show the exact draft version selected for Submission.
- [ ] Request changes can open the corresponding Project in Studio.
- [ ] Marketplace can open a Product in Launcher using identifier-only deep link.
- [ ] Launcher re-authenticates before download/run; deep link contains no token.
- [ ] Launcher and Studio show Runtime compatibility using the same version rules as Backend.
- [ ] Status labels and error codes have one shared glossary across platforms.
- [ ] User-visible timestamps use local timezone while API values remain unambiguous UTC timestamps.

---

# 7. Explicitly deferred from the first MVP UI

- Payment gateway production screens.
- Refund, coupon, affiliate and publisher payout.
- Rating/review and community features.
- Recommendation and personalized homepage.
- Publisher teams and realtime collaboration.
- Visual revision diff.
- Arbitrary Code Node editor.
- Advanced scheduling and remote execution control.
- Multi-PC fleet management.
- Full operational metrics dashboard.
