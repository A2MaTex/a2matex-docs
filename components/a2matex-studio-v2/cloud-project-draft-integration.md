# A2MaTeX Studio — Project Management, Local Autosave & Cloud Sync

**Version:** 2.1

**Architecture:** Local-first / SQLite Autosave / Explicit Cloud Sync

## Overview

Studio tạo project và lưu working copy vào SQLite. Khi đăng nhập, Studio tải danh sách cloud project của Publisher để kiểm tra tên trước khi tạo project. User chủ động nhấn Sync để lưu Cloud Draft, kể cả khi workflow chưa hoàn chỉnh hoặc chưa hợp lệ. Submission là bước riêng có kiểm tra và scan. MVP demo dùng một tài khoản trên một thiết bị tại một thời điểm.

```mermaid
flowchart LR
    A[Studio tạo project] --> B[SQLite autosave]
    B -->|User nhấn Sync| C[Backend lưu Cloud Draft]
    C -->|Trùng tên hoặc xung đột version| D[Hiển thị lỗi và giữ bản local]
    C -->|Submit riêng| E[Submission kiểm tra và scan]
```

## 1. Nguyên tắc thiết kế

A2MaTeX Studio sử dụng mô hình local-first.

- SQLite là source of truth của project working copy.
- Khi tạo project, hệ thống khởi tạo project và draft local trong SQLite.
- Mọi thay đổi trong editor được tự động lưu vào SQLite thông qua debounce.
- Autosave chỉ lưu local, tuyệt đối không tự động gọi Cloud Sync.
- Backend Project quản lý remote identity, ownership và metadata.
- Backend ProjectDraft lưu current workflow snapshot có version.
- Chỉ khi user nhấn nút Sync mới đưa SQLite draft lên cloud.
- Cloud refresh không được tự động ghi đè workflow đã có local.
- Nếu local và cloud có thay đổi không tương thích, dừng Sync, hiển thị lỗi và giữ working copy local.
- Không cho tạo project trùng normalized name.

## 2. Phân biệt Local Draft và Cloud Draft

### 2.1. Local Draft — SQLite

Local Draft là working snapshot được lưu trong SQLite.

Bao gồm:

- Project metadata.
- Workflow documents.
- Input schema.
- Runtime IR.
- Validation summary.
- Test summary.
- Local revision và trạng thái sync.

Local Draft được tạo ngay khi project được tạo.

Sau đó được cập nhật tự động thông qua debounce autosave mỗi khi user chỉnh sửa.

Local Draft không sử dụng backend ProjectDraft `version` làm revision của mình.

### 2.2. Cloud Draft — Backend

Cloud ProjectDraft chỉ tồn tại sau lần Sync thành công đầu tiên.

Cloud Draft chứa:

- `editorGraph`
- `inputSchema`
- `runtimeIr`
- `validationSummary`
- `testSummary`
- `version`
- `contentHash`
- `savedAt`

Mỗi Project có tối đa một current ProjectDraft trên backend.

Mỗi lần sync thành công, backend quyết định version mới theo optimistic locking và same-content retry rules.

### 2.3. So sánh

| Đặc điểm          | Local Draft        | Cloud Draft             |
| ----------------- | ------------------ | ----------------------- |
| Storage           | SQLite             | Backend                 |
| Được tạo khi      | Create Project     | Sync lần đầu            |
| Được cập nhật khi | Autosave           | User nhấn Sync          |
| Revision          | `local_revision`   | `version`               |
| Hoạt động offline | Có                 | Không                   |
| Conflict handling | Local transaction  | Optimistic locking      |
| Source of truth   | Local working copy | Remote current snapshot |

**Invariant:** Autosave không đồng nghĩa với Sync. Một project có thể có hàng trăm lần autosave local nhưng vẫn chưa có Cloud ProjectDraft.

---

## 3. Create Project Lifecycle

### 3.1. Tạo project mới

Khi user submit Create Project:

1. Normalize project name bằng `lower(trim(name))`.
2. Kiểm tra duplicate name trong SQLite và cloud project index của Publisher đã tải khi khởi động. Khi offline, dùng danh sách cloud gần nhất đã cache.
3. Khởi tạo `local_id`.
4. Đặt `remote_project_id = NULL`.
5. Tạo initial Project metadata.
6. Tạo initial WorkflowDocument.
7. Tạo `project_draft_state`.
8. Commit tất cả trong một SQLite transaction.
9. Mở project workspace bằng `local_id`.

Ngay sau transaction, project đã có một local draft hợp lệ, kể cả khi user chưa chỉnh sửa workflow.

Không gọi `POST /projects` hoặc `PUT /draft` trong Create flow.

### 3.2. Giá trị khởi tạo

Ví dụ:

- `local_revision = 0`
- `metadata_revision = 0`
- `remote_project_id = NULL`
- `base_remote_version = NULL`
- `observed_remote_version = NULL`
- `last_synced_local_revision = NULL`
- `sync_status = local-only`

Project có thể được chỉnh sửa và autosave hoàn toàn offline.

---

## 4. Debounced Local Autosave

### 4.1. Mục tiêu

User không cần nhấn Ctrl+S sau mỗi lần chỉnh sửa.

Khi user thay đổi project hoặc workflow, Studio tự động lưu working state vào SQLite sau một khoảng thời gian không có thay đổi mới.

**Debounce đề xuất: 3 giây.**

Thời gian được tính từ lần thay đổi dữ liệu cuối cùng, không phải từ lần click chuột hoặc di chuyển con trỏ cuối cùng.

### 4.2. Autosave trigger

Autosave được kích hoạt khi thay đổi:

- Project name hoặc description.
- Workflow nodes/edges.
- Node position hoặc persisted layout.
- Node parameters.
- Workflow metadata.
- Entry workflow.
- Input schema.
- Các cấu hình khác thuộc persisted authoring document.

Không trigger autosave khi chỉ thay đổi transient UI state như:

- Node selection.
- Hover.
- Open panel.
- Cursor position.
- Temporary viewport state không thuộc document.
- Dữ liệu hiển thị tạm thời.

### 4.3. Debounce behavior

Khi xảy ra thay đổi hợp lệ:

1. Update Zustand working state.
2. Tăng workingRevision.
3. Đánh dấu editor dirty.
4. Reset debounce timer.
5. Nếu không có thay đổi mới trong 3 giây, bắt đầu autosave.
6. Capture immutable snapshot của editor.
7. Kiểm tra snapshot có thể lưu và mở lại. Vẫn autosave khi workflow chưa hoàn chỉnh hoặc không vượt qua validation.
8. Gửi IPC command đến Electron Main.
9. Electron Main ghi SQLite trong một transaction.
10. Sau khi commit, cập nhật savedRevision.

Nếu user tiếp tục chỉnh sửa trong thời gian debounce, timer được reset.

Nếu user chỉnh sửa liên tục, bổ sung cơ chế `maxWait`, ví dụ 30 giây, để tránh trường hợp autosave bị trì hoãn vô hạn.

### 4.4. SQLite transaction

Mỗi autosave commit đồng bộ:

- `projects` metadata.
- Workflow aggregates liên quan.
- `project_draft_state` local fields.

Chỉ tăng `local_revision` một lần cho mỗi transaction có thay đổi dữ liệu thực sự.

Nếu metadata thay đổi, tăng `metadata_revision`.

Nếu không có semantic changes so với snapshot đã lưu, bỏ qua transaction tăng revision không cần thiết.

Nếu graph/input thay đổi làm kết quả cũ hết hiệu lực, đặt `runtimeIr = null` khi chưa có kết quả compile cho snapshot hiện tại và xóa các summary đã stale bằng `{}`. Có thể giữ diagnostics được tạo cho snapshot hiện tại.

Nếu transaction thất bại:

- Không cập nhật savedRevision.
- Giữ editor dirty.
- Hiển thị local save error.
- Cho phép retry.

### 4.5. Race condition

Autosave phải xử lý đúng khi user tiếp tục edit trong lúc SQLite đang ghi.

Ví dụ:

- Editor đạt workingRevision 5.
- Autosave capture revision 5.
- Trong lúc transaction chạy, user edit thành revision 6.
- SQLite commit revision 5 thành công.

Kết quả:

- `savedRevision = 5`
- `workingRevision = 6`
- Editor vẫn dirty.
- Revision 6 được autosave trong lần tiếp theo.

Không đánh dấu toàn bộ editor clean chỉ vì một autosave cũ đã thành công.

Các write transaction của cùng project phải được serialize hoặc điều phối để snapshot cũ không ghi đè snapshot mới.

### 4.6. Flush pending autosave

Khi xảy ra:

- User nhấn Sync.
- User chuyển project.
- User đóng workspace.
- User nhấn Ctrl+S.

Studio phải cố gắng flush pending changes xuống SQLite.

Đặc biệt với Sync, local commit phải hoàn tất thành công trước khi đọc snapshot để upload.

Nếu local save thất bại, chặn Sync để không upload một bản draft cũ ngoài ý muốn.

Khi application bị force-kill hoặc hệ điều hành crash, không thể bảo đảm flush. Vì vậy debounce và maxWait cần được thiết kế để giảm cửa sổ mất dữ liệu.

---

## 5. Recent Projects — Local First

Khi khởi động Studio với tài khoản đã đăng nhập, gọi `GET /projects` và lấy đủ các trang thuộc Publisher hiện tại để cập nhật cloud project index. Mỗi khi truy cập vào màn hình chính của studio hiển thị danh sách project, gọi lại api này một lần nữa để đảm bảo luôn có danh sách cloud projects mới nhất và ko gặp name conflict khi tạo project mới. Khi offline, dùng index đã cache cho tài khoản đó nếu có.

Khi mở Recent Projects:

1. Query danh sách SQLite.
2. Render project local ngay lập tức.
3. Dùng cloud project index và làm mới qua `GET /projects` khi cần.
4. Merge cloud projects theo `remote_project_id`.
5. Thêm cloud-only projects chưa có trong SQLite.
6. Giữ nguyên local working copy.
7. Sort theo thời gian cập nhật gần nhất.

`effectiveUpdatedAt = max(local_updated_at, remote_updated_at)`

Local project chưa Sync vẫn hiển thị trong Recent Projects.

Cloud-only project được hiển thị nhưng chưa cần materialize vào SQLite cho đến khi user mở.

Nếu local-only và cloud-only trùng normalized name mà chưa có remote binding, hiển thị name-conflict và không tự merge theo tên.

---

## 6. Explicit Cloud Sync

### 6.1. Nguyên tắc

Nút **Sync** là entry point duy nhất để upload project/draft lên backend trong normal authoring flow.

Autosave không tự gọi Sync.

Sync phải lấy dữ liệu từ SQLite, không trực tiếp lấy từ Zustand.

### 6.2. Khi user nhấn Sync

Thực hiện theo thứ tự:

1. Disable hoặc serialize thao tác Sync trùng nhau trên cùng project.
2. Flush pending debounce autosave.
3. Chờ SQLite transaction hoàn tất.
4. Read immutable SQLite snapshot ở local revision N.
5. Resolve remote project identity.
6. GET Project/Draft nếu cần để xác định trạng thái remote.
7. Detect version/name conflict.
8. Nếu trùng tên hoặc có xung đột version, dừng Sync, hiển thị lỗi và giữ bản local.
9. Nếu không conflict, đồng bộ metadata.
10. Build ProjectDraft payload từ SQLite snapshot, kể cả khi workflow chưa hoàn chỉnh hoặc không hợp lệ. Dùng `runtimeIr: null` nếu chưa có IR cho snapshot hiện tại. Kiểm tra điều kiện submit và scan ở bước Submission.
11. PUT Cloud Draft.
12. Persist remote acknowledgement về SQLite.
13. Cập nhật sync status.

### 6.3. Case A — Project chưa có trên cloud

Điều kiện:

`remote_project_id = NULL`

Flow:

1. Làm mới cloud project index của Publisher và kiểm tra tên trước khi gọi `POST /projects`.
2. Nếu tên đã tồn tại và Studio chưa xác nhận binding, dừng Sync, giữ bản local và yêu cầu user đổi tên. Nếu lần tạo trước bị mất phản hồi, áp dụng cùng quy tắc cho Project trùng tên chưa có binding.
3. Gọi `POST /projects` với metadata từ SQLite và lưu ID trả về làm binding. Trong MVP một tài khoản dùng một thiết bị tại một thời điểm, Studio serialize bước kiểm tra tên và tạo cloud Project cho cùng Publisher. Backend có thể trả ID của Project trùng tên, kể cả khi Project đó chưa có draft, nên flow này dựa trên danh sách cloud vừa tải và giả định chỉ có một thiết bị ghi.
4. Với binding đã xác nhận, kiểm tra Cloud Draft. Khi `draft: null`, gửi PUT với `expectedVersion = 0`.
5. Lưu version, hash và savedAt do Backend trả về vào SQLite.

Nếu POST đã tạo Project và binding đã được lưu nhưng Draft PUT thất bại, lần Sync sau dùng binding đó để tiếp tục.

### 6.4. Case B — Project đã có trên cloud

Điều kiện:

`remote_project_id != NULL`

Flow:

1. GET remote Project/Draft.
2. Kiểm tra project status và quyền.
3. So sánh remote version/hash với local baseline.
4. Nếu remote không thay đổi, cho phép push local draft.
5. Nếu remote thay đổi và local có thay đổi chưa Sync, hiển thị conflict và giữ bản local. Nếu đang retry đúng payload của lần PUT bị mất phản hồi, gửi lại với expectedVersion ban đầu để Backend xác nhận cùng content hoặc trả conflict.
6. Nếu remote thay đổi và local clean, chuyển remote-ahead, không auto pull.
7. Khi push, sử dụng expectedVersion phù hợp.
8. Nhận remote acknowledgement.
9. Update `last_synced_local_revision = N`.

Nếu trong lúc Sync, editor đã autosave local revision N+1, trạng thái vẫn là `local-ahead`.

### 6.5. Metadata Sync

Project metadata được sync qua Project API riêng.

Workflow được sync qua ProjectDraft API.

Không đưa Project name, description, publisher hoặc sync metadata vào `editorGraph`.

Nếu metadata update thành công nhưng Draft PUT thất bại, giữ metadata acknowledgement và retry phần Draft còn pending.

---

## 7. Local và Cloud Version Management

### 7.1. Revision fields

| Field                        | Vai trò                            |
| ---------------------------- | ---------------------------------- |
| `workingRevision`            | Editor working state               |
| `savedRevision`              | Revision đã autosave local         |
| `local_revision`             | SQLite durable revision            |
| `metadata_revision`          | Metadata local revision            |
| `metadata_synced_revision`   | Metadata đã sync                   |
| `base_remote_version`        | Cloud baseline                     |
| `observed_remote_version`    | Cloud version mới nhất đã quan sát |
| `last_synced_local_revision` | SQLite revision đã upload          |

### 7.2. Ví dụ lifecycle

| Action            | Local revision | Cloud draft version |
| ----------------- | -------------: | ------------------- |
| Create Project    |              0 | NULL                |
| Edit + Autosave   |              1 | NULL                |
| Edit + Autosave   |              2 | NULL                |
| Edit + Autosave   |              3 | NULL                |
| Nhấn Sync lần đầu |              3 | 1                   |
| Edit + Autosave   |              4 | 1                   |
| Edit + Autosave   |              5 | 1                   |
| Nhấn Sync lần hai |              5 | 2                   |

Các version cloud trong ví dụ giả định mỗi lần Sync có content mới và backend tăng version một lần.

**Local revision và Cloud Draft version là hai bộ đếm độc lập.**

Autosave không làm tăng Cloud Draft version.

Nếu Sync lại cùng content, backend có thể trả success với version không đổi.

---

## 8. Conflict và bảo vệ Local Draft

Cloud refresh không tự thay workflow đang có trong SQLite, dù local draft đã sync hay chưa.

Nếu remote draft thay đổi:

- Local clean: hiển thị remote-ahead, cho phép user chọn cập nhật.
- Local dirty hoặc local-ahead: chuyển conflict.
- Không auto import remote editorGraph.
- Không tự chọn cloud-wins.

Nếu Backend trả version conflict, Studio hiển thị lỗi và giữ working copy trong SQLite. Các thao tác review, ghi đè cloud và thay bản local bằng bản cloud thuộc phạm vi hỗ trợ đa thiết bị sau MVP.

---

## 9. UI States

Tách trạng thái Local Save và Cloud Sync.

### Local Save State

- `dirty`: Editor có thay đổi chưa commit.
- `saving-local`: SQLite autosave đang chạy.
- `saved-local`: Working copy đã persist.
- `local-save-error`: Local commit thất bại.

### Cloud Sync State

- `local-only`: Chưa có remote project.
- `local-ahead`: SQLite có thay đổi chưa Sync.
- `syncing`: Đang Sync.
- `synced`: Local revision hiện tại đã Sync.
- `remote-ahead`: Cloud có version mới.
- `conflict`: Cần xử lý khác biệt.
- `name-conflict`: Trùng tên, chưa xác định binding.
- `offline`: Cloud unavailable.
- `error`: Cloud Sync thất bại.

### UI Behavior

Khi user chỉnh sửa:

`Unsaved changes → Saving locally… → Saved locally`

Nếu chưa Sync, vẫn hiển thị trạng thái cloud riêng như:

`Not synced` hoặc `Local changes`

Khi user nhấn Sync và backend xác nhận:

`Syncing… → Synced`

Nếu local thay đổi thêm trong lúc Sync:

`Syncing… → Local changes`

Không hiển thị `Synced` khi chỉ autosave SQLite thành công.

---

## 10. Definition of Done

- Create Project tạo SQLite project và initial Local Draft trước khi mở workspace.
- Debounce autosave mặc định 3 giây sau lần thay đổi dữ liệu cuối cùng.
- Autosave chỉ ghi SQLite, không gọi Project/Draft mutation API.
- Local revision tăng khi có semantic changes được commit.
- Metadata và workflow được save atomic.
- Autosave đang chạy không làm mất các edit mới hơn.
- Sync luôn flush pending autosave trước khi đọc SQLite snapshot.
- Sync chỉ upload dữ liệu từ immutable SQLite snapshot.
- Project chưa có cloud được tạo remote identity trước khi tạo ProjectDraft.
- Cloud Draft đầu tiên dùng `expectedVersion = 0` khi remote draft chưa tồn tại.
- Project đã có cloud phải kiểm tra remote version/hash trước khi push.
- Remote version và contentHash được lấy từ backend acknowledgement.
- Recent Projects render local trước, merge cloud sau.
- Không cho tạo duplicate normalized name theo local và cloud đã biết.
- Không tự overwrite local workflow khi remote thay đổi.
- Studio tải cloud project index của Publisher khi khởi động và dùng cache khi offline nếu có.
- First Sync gặp tên trùng khi chưa xác nhận binding thì giữ bản local và yêu cầu đổi tên.
- Sync gặp version conflict thì hiển thị lỗi và giữ bản local.
- Workflow chưa hoàn chỉnh hoặc không hợp lệ vẫn được autosave và Sync. IR và summary hết hiệu lực được xử lý trước khi upload. Submission thực hiện kiểm tra và scan riêng.
- Network failure không làm mất Local Draft.
- Cloud sync chỉ diễn ra khi user chủ động nhấn Sync.
- Một project có thể autosave local nhiều lần mà Cloud Draft version không đổi.

**Final invariant:** Create tạo Local Draft. Edit kích hoạt Debounced Autosave vào SQLite. Sync mới đưa Local Draft lên Backend ProjectDraft và nhận Cloud Draft version. Cloud refresh không tự thay thế Local Draft.
