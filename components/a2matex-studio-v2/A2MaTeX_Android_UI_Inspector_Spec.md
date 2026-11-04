# A2MaTeX Studio — Android UI Inspector

> **Status:** Proposed specification · **Version:** 1.0 · **Audience:** Product, UI/UX, Electron/React, Go Runtime, QA, coding agents
>
> **Scope:** Snapshot-based inspection of Android UI elements in A2MaTeX Studio. This document is the source of truth for MVP behavior, interface contracts, implementation boundaries, and acceptance tests.

## 1. Overview

The Android UI Inspector helps Publishers select Android UI elements while authoring automation workflows. The Inspector captures a **manual snapshot** of a connected Android device, displays its screenshot, overlays UI element bounds extracted from the accessibility hierarchy, and allows users to click a location to inspect **all elements whose bounds contain that point**.

The Inspector is **not** a live mirror, a web-style DOM inspector, or an Android interaction emulator. It does not continuously poll or refresh. The user explicitly presses **Inspect** to capture a new snapshot.

### 1.1 Goals

- Select a connected Android device by its ADB serial (`deviceId`).
- Capture screenshot, UI hierarchy, and display metadata on demand.
- Render screenshot with accurate coordinate mapping and element overlays.
- On click, list overlapping candidate elements, with deterministic ordering.
- Show essential metadata and allow copying resource ID or a structured selector.
- Allow transferring a selector to a compatible workflow node.
- Keep Studio's design-time selector authoring separate from Go Runtime's execution-time selector resolution.

### 1.2 Non-goals (MVP)

- Real-time video, polling, or continuous hierarchy updates.
- Guaranteeing access to every element in games, Canvas-based UIs, or WebViews.
- Guaranteeing that a resource ID is unique or stable across app versions.
- Automatically executing clicks or text input when an element is selected in Studio.
- Full Android accessibility debugging or native Z-order reconstruction.

## 2. Terminology and constraints

| Term | Definition |
|---|---|
| Device ID | ADB serial identifying a connected emulator or device, e.g. `emulator-5554`. |
| Snapshot | One capture containing screenshot, hierarchy, metadata, timestamp, and device identity. |
| UI element | Node parsed from the Android UI hierarchy XML. |
| Bounds | Element rectangle in Android screenshot coordinate space. |
| Candidate | Element whose bounds contain the clicked screenshot point. |
| Selector | Portable criteria used to identify a UI element at workflow execution time. |
| Resolve | Find matching element(s) in the **current** Android UI and determine actionable coordinates. |

**Important constraints:** Resource IDs may be absent, duplicated (e.g. repeated RecyclerView rows), or changed between app versions. XML hierarchy does not reliably expose actual visual Z-order. Screenshot and hierarchy are captured sequentially and may be inconsistent if the UI changes. Some apps expose partial or no useful hierarchy.

## 3. User journeys

### 3.1 Open and capture

1. Publisher opens the **Android Inspector** dock on the left side of Studio's workflow editor.
2. Studio requests ADB device enumeration from Electron Main.
3. A device selector shows serial, model, and connection status. Only usable `device` entries are selectable.
4. Publisher selects a device and presses **Inspect**.
5. Studio disables duplicate capture requests for the selected device and displays progress.
6. Electron Main captures a PNG screenshot, UI hierarchy XML, and display metadata; it returns a normalized snapshot.
7. Studio displays the screenshot at an aspect-preserving scale, along with snapshot timestamp and capture status.

### 3.2 Click to inspect

1. Publisher clicks a point on the rendered screenshot.
2. Renderer converts the pointer position to **original screenshot pixel coordinates**.
3. Hit-testing finds every node whose bounds contain that point.
4. Candidates are sorted using the deterministic rules in §7.
5. Inspector shows the complete candidate list (scrollable if needed), with the first candidate selected by default.
6. Selecting a candidate highlights its bounds on the screenshot and expands its details.
7. Publisher copies its resource ID, copies a selector, or chooses **Use in Node** if the workflow node supports the selector.

### 3.3 Refresh

Pressing **Inspect** again replaces the current snapshot, clears stale candidate selection and highlight, and updates the timestamp. The selected device remains unchanged. A failed refresh preserves the last successful snapshot but marks it **Outdated / last successful capture**, with an actionable error message.

### 3.4 Change device

Changing the device clears the previous device's screenshot and selection to avoid accidentally transferring a selector from the wrong device. The user must press Inspect to capture the new device.

## 4. Detailed UI/UX specification

### 4.1 Layout

```
┌────────────────────────────────┬──────────────────────────────────────┐
│ ANDROID INSPECTOR (left dock)  │ WORKFLOW CANVAS                      │
│ [Device ▼] [Refresh devices]   │                                      │
│ [Inspect]  Snapshot 10:25:31   │   [Input Text Node]                  │
│                                │      Selector: [ ... ]               │
│ ┌────────────────────────────┐ │                                      │
│ │                            │ │                                      │
│ │  Scaled Android screenshot │ │                                      │
│ │  [selected bounds outline] │ │                                      │
│ │                            │ │                                      │
│ └────────────────────────────┘ │                                      │
│ Clicked point: (x, y)          │                                      │
│ Elements at this point (3)     │                                      │
│ ① TextView           [selected]│                                      │
│ ② Button                       │                                      │
│ ③ LinearLayout                 │                                      │
│ ────────────────────────────── │                                      │
│ Package: com.example.app       │                                      │
│ Resource ID: .../login  [Copy] │                                      │
│ Class: android.widget.TextView │                                      │
│ Text: Login                    │                                      │
│ Content description: —         │                                      │
│ Bounds: [120,450][960,530]     │                                      │
│ Clickable: false               │                                      │
│ [Copy selector] [Use in Node]  │                                      │
└────────────────────────────────┴──────────────────────────────────────┘
```

The left dock should be resizable and collapsible. The screenshot preserves its original aspect ratio and must never be stretched to fill a nonmatching container. Long values wrap or truncate with a copy action. Candidate list and details are scrollable independently where feasible.

### 4.2 Device selector

- Label: **Android Device**.
- Show `model — serial` and status (`device`, `offline`, `unauthorized`).
- Include **Refresh devices** action.
- Disable Inspect if no usable device is selected.
- If device disconnects, show **Device disconnected**; do not silently switch to another device.

### 4.3 Capture controls

- Primary button: **Inspect**; during capture: **Inspecting…** with loading indicator.
- Display **Snapshot captured at HH:mm:ss** (local time), device serial, and optionally screenshot dimensions.
- Explicitly indicate snapshot is static: **Refresh to inspect the current screen**.
- Do not auto-refresh on mouse movement, node selection, window focus, or timer.

### 4.4 Screenshot interaction

- Pointer over screenshot: normal pointer or crosshair; **no hover-driven metadata popup**.
- Click: select point and compute candidates.
- Selected element: visible outline and translucent overlay aligned to its bounds; avoid covering the underlying UI completely.
- If candidates overlap, changing selected candidate updates the highlight immediately without recapturing.
- If no candidate matches: show **No inspectable element at this position** and optionally allow copying coordinates (future enhancement).
- Click outside the actual displayed screenshot (letterboxed region): do nothing.

### 4.5 Candidate list

- Heading: **Elements at selected position (N)**.
- Each row: ordinal, short class name, resource ID (or **No resource ID**), optional depth.
- Default sort: deeper nodes first, then smaller bounds area, then original XML traversal index.
- This is a **heuristic**, not a guarantee of actual visual front-to-back order; UI should say **Most specific first**, not **Actual Z-order**.
- The user can select any candidate. All candidates remain available even if some have no resource ID.

### 4.6 Element detail fields

| Field | Required in UI | Behavior |
|---|---|---|
| Package name | Yes | Extract from node `package`; show `—` if missing. |
| Resource ID | Yes | Show full ID, or `—`; disable resource-ID copy if missing. |
| Class name | Yes | Full Android class name. |
| Text | Yes | Display if available; avoid logging sensitive user text. |
| Content description | Yes | Display if available. |
| Bounds | Yes | Original screenshot pixel rectangle. |
| Clickable | Yes | Boolean from hierarchy. |
| Enabled | Recommended | Boolean from hierarchy. |
| Depth / node path | Recommended | Useful for overlapping nodes and debugging. |

Copy actions: **Copy Resource ID** copies the raw ID string; **Copy Selector** copies JSON with a supported strategy and value; **Use in Node** applies the selector to the currently selected compatible workflow node. Disable Use in Node if no compatible node is selected. Do not invent a resource ID for nodes lacking one; a later version may support `contentDescription`, text, or XPath strategies.

### 4.7 UI states

| State | Expected UI |
|---|---|
| No devices | Empty device selector, explanation, Refresh devices. |
| Unauthorized/offline | Device listed but disabled with guidance. |
| Ready, no snapshot | Empty screenshot placeholder: **Select a device and press Inspect**. |
| Capturing | Loading indicator; prevent duplicate capture. |
| Snapshot ready | Screenshot, timestamp, click instruction. |
| No click yet | Candidate panel: **Click an element in the screenshot**. |
| No candidates | Informational empty state; no selected element. |
| Candidates found | Sorted list, first item selected, highlight and details. |
| Partial hierarchy | Screenshot still visible; warning that some UI elements may be unavailable. |
| Capture failure | Error and Retry; retain last successful snapshot if same device, marked stale. |
| Device disconnected | Warning, disable Inspect, offer Refresh devices. |

## 5. Architecture and ownership

```
Studio React Renderer
  ├─ InspectorPanel (device selection, screenshot, candidates, details)
  ├─ ScreenshotViewport (pixel mapping and bounds overlay)
  └─ WorkflowEditor (selector binding)
           │ typed IPC via preload
           ▼
Electron Main
  ├─ InspectorService (capture orchestration, timeout, cancellation)
  ├─ AdbService (device enumeration and shell/exec-out)
  ├─ UiHierarchyParser (XML → normalized nodes)
  └─ SnapshotAssembler (image + metadata + nodes)
           │
           ▼
Android device via ADB

Published workflow JSON
           │
           ▼
Launcher → Go Runtime
  ├─ SelectorResolver (query CURRENT hierarchy)
  └─ ActionExecutor (tap/type/etc. after resolution)
```

**Design-time vs runtime:** Studio creates and validates selector configurations using a snapshot. Go Runtime must re-query the live Android UI when the workflow executes. Do **not** persist design-time bounds as the authoritative click target. Bounds may optionally be retained as debugging metadata only.

## 6. ADB capture procedure

Example commands (always pass selected serial with `-s`):

```bash
adb devices -l
adb -s emulator-5554 exec-out screencap -p
adb -s emulator-5554 shell wm size
adb -s emulator-5554 shell uiautomator dump /sdcard/a2matex-window.xml
adb -s emulator-5554 exec-out cat /sdcard/a2matex-window.xml
```

Implementation notes:

1. Invoke ADB using argument arrays (e.g. `spawn`/`execFile`), **not** interpolated shell commands.
2. Capture screenshot as binary PNG; do not decode it as UTF-8.
3. Parse `wm size` output defensively (physical vs override size); the decoded PNG dimensions are the primary screenshot coordinate reference.
4. UIAutomator dump can be slow, incomplete, or fail; apply timeouts and surface clear errors.
5. Screenshot and XML are not atomic. If they mismatch, inform the user and allow recapture.
6. Use unique temporary XML paths per capture if needed, clean up after use, and avoid concurrent captures on the same device.
7. Limit image/XML payload sizes and release old object URLs and buffers on refresh or unmount.
8. Do not require persistent polling, accessibility-service installation, or root privileges for MVP.

## 7. Coordinate mapping and overlapping-node algorithm

### 7.1 Screenshot geometry

Given screenshot dimensions `imageW × imageH` and viewport dimensions `viewW × viewH`, calculate:

```ts
const scale = Math.min(viewW / imageW, viewH / imageH);
const renderedW = imageW * scale;
const renderedH = imageH * scale;
const offsetX = (viewW - renderedW) / 2;
const offsetY = (viewH - renderedH) / 2;

const imageX = (pointerX - offsetX) / scale;
const imageY = (pointerY - offsetY) / scale;
```

`pointerX/Y` must be measured relative to the screenshot viewport, not the window. Ignore clicks outside the rendered image. If display rotation, cropping, or screenshot/hierarchy dimensions differ, normalize into the same coordinate space before hit-testing; never silently assume identical geometry.

Render a node overlay using:

```ts
const left = offsetX + bounds.left * scale;
const top = offsetY + bounds.top * scale;
const width = (bounds.right - bounds.left) * scale;
const height = (bounds.bottom - bounds.top) * scale;
```

### 7.2 Hit-testing

Use half-open rectangles (`left <= x < right`, `top <= y < bottom`). Ignore malformed or zero-area bounds.

```ts
function getCandidates(nodes: InspectorNode[], x: number, y: number) {
  return nodes
    .filter(n => n.bounds.left <= x && x < n.bounds.right &&
                 n.bounds.top <= y && y < n.bounds.bottom)
    .sort((a, b) =>
      b.depth - a.depth ||
      area(a.bounds) - area(b.bounds) ||
      a.traversalIndex - b.traversalIndex
    );
}
```

`depth` and `traversalIndex` come from the parsed XML tree. Sorting is deterministic and prioritizes specificity; it is not a true Android Z-order computation. Preserve the full candidate list, including parents and nodes without resource IDs.

### 7.3 Selection and highlighting

On each click, store `{ snapshotId, imagePoint, candidateNodeIds, selectedNodeId }`. On candidate selection, update only `selectedNodeId` and overlay. On snapshot refresh or device change, clear selection. No new ADB call is needed for candidate switching.

## 8. Suggested TypeScript contracts

```ts
type Bounds = { left: number; top: number; right: number; bottom: number };

type InspectorNode = {
  nodeId: string;                // unique within snapshot, NOT across snapshots
  parentId?: string;
  traversalIndex: number;
  depth: number;
  packageName?: string;
  resourceId?: string;
  className: string;
  text?: string;
  contentDescription?: string;
  bounds: Bounds;
  clickable: boolean;
  enabled: boolean;
};

type InspectorSnapshot = {
  snapshotId: string;
  deviceId: string;
  capturedAt: string;           // ISO 8601
  screenshot: { mimeType: 'image/png'; width: number; height: number; dataUrl?: string };
  screen: { width?: number; height?: number; rotation?: number };
  nodes: InspectorNode[];
  warnings: string[];
};

type Selector =
  | { strategy: 'resourceId'; value: string; packageName?: string }
  | { strategy: 'contentDescription'; value: string; packageName?: string };
```

For large screenshots, prefer a binary-safe IPC transport or temporary local asset protocol over embedding large base64 strings. The `dataUrl` field above is optional, not a requirement.

### 8.1 Suggested preload API

```ts
window.a2matexInspector = {
  listDevices: (): Promise<AndroidDevice[]> => ipcRenderer.invoke('inspector:list-devices'),
  capture: (deviceId: string): Promise<InspectorSnapshot> =>
    ipcRenderer.invoke('inspector:capture', { deviceId }),
};
```

Expose only validated operations through preload. Do not expose raw shell execution to the renderer. Validate device IDs against the enumerated device list in Main and validate IPC payload sizes.

### 8.2 Workflow node binding example

```json
{
  "type": "tapElement",
  "config": {
    "selector": {
      "strategy": "resourceId",
      "value": "com.example.app:id/login_button",
      "packageName": "com.example.app"
    },
    "timeoutMs": 10000
  }
}
```

The selector contract must be versioned and supported consistently by Studio, Launcher, and Go Runtime. Do not include a snapshot-specific `nodeId` as a runtime selector.

## 9. Runtime resolution contract (separate feature boundary)

When a workflow executes, Go Runtime:

1. Receives the node's selector configuration.
2. Queries the **current** UI hierarchy on the assigned Android device.
3. Matches candidates according to selector strategy and optional package scope.
4. Applies explicit handling for **zero**, **one**, or **multiple** matches.
5. For a unique actionable match, obtains current bounds and performs the action (e.g. tap center).
6. Emits structured execution status/errors to Launcher.

Suggested outcomes: `ELEMENT_FOUND`, `ELEMENT_NOT_FOUND`, `AMBIGUOUS_SELECTOR`, `ELEMENT_NOT_ACTIONABLE`, `UI_DUMP_FAILED`, `DEVICE_DISCONNECTED`, `TIMEOUT`.

A duplicated resource ID must **not** silently resolve to the first match. For MVP, fail with `AMBIGUOUS_SELECTOR` and require a more specific selector or explicit disambiguation in a later release. Retry/wait semantics should be node-configurable and bounded by a timeout. Studio must not promise that a selector is valid on all devices or future app versions.

## 10. Error handling and security

- ADB unavailable: show setup/troubleshooting guidance and Refresh devices.
- Device unauthorized: instruct user to accept Android USB debugging authorization.
- UI dump failure: show screenshot if available and an explicit hierarchy warning; do not render fabricated overlays.
- Malformed XML: fail parsing safely; never execute XML contents.
- Screenshot decode failure: show capture error and allow retry.
- Device disconnected mid-capture: abort or fail cleanly, without showing a misleading successful snapshot.
- Long-running capture: timeout and cancel/terminate child processes as needed.
- Avoid persisting screenshots, text fields, or UI dumps by default. They may contain personal information or secrets.
- Never log passwords, sensitive UI text, raw screenshots, or complete XML dumps in production telemetry.
- Validate selector data at workflow save, publish, and runtime boundaries.

## 11. Performance and resource management

- Capture **only** on explicit Inspect action; no polling.
- Reuse the existing ADB executable/service where practical.
- Keep one active snapshot per Inspector panel; dispose old screenshot object URLs and references on refresh.
- Avoid rendering one interactive DOM overlay per hierarchy node if large trees cause lag: perform hit-testing on in-memory node data and render only selected highlight(s).
- Memoize parsed node data and geometry where helpful; recompute geometry on panel resize without recapturing.
- Add bounded timeouts and maximum XML/image sizes; benchmark on low-end Windows devices.

## 12. MVP acceptance criteria

| ID | Given / When / Then |
|---|---|
| AC-01 | Given two connected devices, when opening device selector, then both are listed by serial and model. |
| AC-02 | Given a usable selected device, when clicking Inspect, then a screenshot and hierarchy snapshot are captured once without polling. |
| AC-03 | Given a captured snapshot, when resizing the dock, then screenshot and highlight remain correctly aligned. |
| AC-04 | Given a click inside multiple nested bounds, then all matching candidates are listed in deterministic most-specific-first order. |
| AC-05 | Given a candidate selection, then only its highlight and details change, without any ADB request. |
| AC-06 | Given a candidate with a resource ID, when clicking Copy Resource ID, then the exact raw value is copied. |
| AC-07 | Given a candidate without a resource ID, then the field displays `—` and resource-ID copy is disabled. |
| AC-08 | Given a compatible selected workflow node, when clicking Use in Node, then its selector config is updated and remains valid after save/reopen. |
| AC-09 | Given a new Inspect capture, then old candidate selection is cleared and snapshot timestamp changes. |
| AC-10 | Given an offline/unauthorized device, then Inspect is disabled or fails with actionable guidance, not an unhandled exception. |
| AC-11 | Given a hierarchy dump failure with valid screenshot, then screenshot is shown with a warning and no invented candidates. |
| AC-12 | Given a duplicated resource ID at runtime, then resolver reports ambiguity rather than clicking an arbitrary element. |
| AC-13 | Given a click in screenshot letterboxing, then no candidate is selected. |
| AC-14 | Given a device change, then the previous device's screenshot and selection are cleared. |

## 13. Implementation milestones

**P0 — Inspector MVP:** ADB device list, manual capture, screenshot viewport, XML parser, coordinate mapping, hit-testing, candidate list, details, highlight, copy resource ID, refresh/error states.

**P1 — Workflow integration:** Typed selector contract, Copy Selector, Use in Node, compatibility validation, runtime resource-ID resolution and explicit ambiguity errors.

**P2 — Advanced inspection:** Additional selector strategies, candidate filtering/search, optional coordinate fallback, hierarchy navigation, and capture diagnostics.

## 14. Agent review checklist

Before implementation or approval, verify:

- [ ] No automatic refresh/polling is introduced.
- [ ] UI interaction is click-to-inspect, not hover-to-inspect.
- [ ] All overlapping candidates are preserved; sorting is labeled heuristic.
- [ ] Package name and resource ID are visible, including missing-value states.
- [ ] Coordinates are based on actual screenshot dimensions and letterbox offsets.
- [ ] Capture failures and screenshot/XML inconsistencies are surfaced.
- [ ] Preload IPC exposes only narrowly scoped, validated functions.
- [ ] No arbitrary ADB shell string is accepted from renderer.
- [ ] Snapshot IDs and bounds are not treated as persistent runtime selectors.
- [ ] Duplicated resource IDs are handled explicitly.
- [ ] Screenshot/element text is not retained or logged unnecessarily.
- [ ] Studio, workflow schema, and Go Runtime agree on selector semantics.

## 15. Open decisions

1. Should P1 `Use in Node` be available only for tap nodes or also text input, wait, and assertion nodes?
2. Should Studio support composite selectors in P1 (resource ID + text / parent context), or wait until P2?
3. Which IPC screenshot transport is preferred: binary bytes, data URL, or controlled local asset protocol?
4. What timeout and size limits are appropriate for the supported device fleet?
5. Should manual coordinate selection be in P1 for apps with inaccessible hierarchy?

**Default recommendation:** Ship the P0 Inspector independently; finalize and version the selector schema before building P1 workflow binding and Go Runtime resolution.
