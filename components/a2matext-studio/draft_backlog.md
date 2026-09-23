# 8. Backlog A2MaTex Studio

## Epic 1 — Desktop foundation

- Khởi tạo Electron + React + TypeScript.
- Thiết lập main/preload/renderer.
- Bật sandbox và context isolation.
- IPC allowlist.
- Local SQLite.
- Project create/open/save.
- Autosave.

## Epic 2 — Workflow canvas

- Node palette.
- Drag/drop node.
- Connect/disconnect edge.
- Custom node UI.
- Properties inspector.
- Zoom/pan/minimap.
- Copy/paste.
- Undo/redo.
- Delete confirmation.
- Persist canvas layout.

## Epic 3 — Workflow domain model

- Project schema.
- Graph schema.
- Editor schema.
- Node definition registry.
- Node versioning.
- Variable schema.
- Expression parser.
- Schema migration.

## Epic 4 — MVP node catalog

- Start/End/Fail.
- LaunchApp.
- WaitForElement.
- TapElement.
- Swipe.
- InputText.
- PressBack.
- Delay.
- Condition.
- Repeat với giới hạn.
- GetElementText.
- ElementExists.

## Epic 5 — Validation and compiler

- Structural validator.
- Flow validator.
- Configuration validator.
- Capability collector.
- Graph normalizer.
- IR compiler.
- Deterministic hash.
- Problems panel.

## Epic 6 — Local simulation

- Mock command result.
- Variable inspector.
- Branch simulation.
- Execution trace.
- Validate before run.
- Stop/reset simulation.

## Epic 7 — Live device debug

- Pair test device.
- Start test session.
- Run/stop.
- Breakpoint.
- Continue.
- Step over.
- Current-node highlight.
- Command/result console.
- Disconnect handling.

## Epic 8 — Cloud draft

- Publisher login.
- Create cloud project.
- Push revision.
- Pull revision.
- Detect revision conflict.
- Manual sync.
- Sync status indicator.

## Epic 9 — Submission preparation

- Submission form.
- Product metadata.
- Capability declaration.
- Input/output definition.
- Compliance questionnaire.
- Asset validation.
- Generate IR.
- Generate checksums.
- Upload immutable submission snapshot.
