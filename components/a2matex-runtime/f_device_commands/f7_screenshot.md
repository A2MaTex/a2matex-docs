# F7 — Capture a full screen or rectangular region

## Overview

Screenshot captures a PNG from the Android device selected for the current
execution. The publisher chooses Full screen, Absolute pixels, or Screen
percentage. Both region modes use a top-left and a bottom-right corner.
ADB captures one full frame. For a region, Runtime crops that frame in
memory and gives D5 only the cropped PNG. The workflow receives an
`artifactId` after D4 commits the successful attempt.

```mermaid
flowchart LR
    I[Selected device and capture mode] --> V[Validate exact fields for the mode]
    V -->|Invalid| X[Return failure]
    V -->|Valid| C[F1 and E1 capture one bounded PNG]
    C -->|Capture or image fails| X
    C -->|Valid frame| R[F7 selects full frame or crops]
    R --> S[D5 stages the selected PNG]
    S -->|Stage fails| X
    S -. Phase G hands staged metadata to D4 .-> D[D4 commits artifact and attempt]
    D -->|Confirmed| O[Workflow receives artifactId]
    D -->|Uncertain| P[D4 reads back commit before publication]
```

**Status** — Runtime validation, bounded capture, all three capture modes,
and the staging callback are implemented. The local smoke runner exercises
the real D4/D5 commit path using the normal Runtime storage location and
prints the database and PNG paths. Stop the Runtime server before this
smoke run so it can acquire the storage lock. Linux automated checks and
Windows cross-compilation pass. Phase G/A7 owns the workflow runner
callback, checkpoint handoff, artifact retrieval API, and shipped F9
registration. Native Windows, remaining live-device acceptance, catalog
publication, and Studio integration remain pending verification.

## 1. Node contract

| Item | Requirement |
|---|---|
| Node type and handler key | Both are `android.screenshot`. |
| Capability | `android.screen.capture`. |
| Config | Exactly one field, `targetMode`. Screenshot declares its own list of allowed modes. |
| Context | The execution supplies the exact ADB serial and attempt deadline. The default node timeout is 30,000 ms. |
| Successful output | `{"artifactId":"<opaque ID>"}`. Width and height describe the stored image in D5 metadata. |
| Control flow | `success` and `error` ports with the existing `STOP` and `ERROR_BRANCH` policies. G applies the admitted policy. |

| `targetMode` | Top-left `(x, y)` | Bottom-right `(x, y)` | Literal bounds |
|---|---|---|---|
| `FULL_SCREEN` | No bindings | No bindings | Empty `bindings` and empty IR `inputs` |
| `ABSOLUTE_PIXELS` | `startX`, `startY` | `endX`, `endY` | Integers from 0 to 4,096 |
| `SCREEN_PERCENT` | `startXPercent`, `startYPercent` | `endXPercent`, `endYPercent` | Integers from 0 to 100 |

Studio builds the dropdown from Screenshot's `editorHints.targeting.modes`,
selects and saves `FULL_SCREEN` for a new node, and renders only the selected
mode's fields. It displays **Top-left** and **Bottom-right** point groups,
with X and Y for each point, and clears previous-mode coordinates when the
mode changes. Any preview uses the same corner order. Runtime uses the
actual captured frame to establish dimensions and orientation.

For `20%, 20%` to `40%, 30%`, Runtime IR carries this config and these
inputs. Editor Graph uses `bindings` and `kind: "LITERAL"` for the same
four values.

```json
{
  "config": { "targetMode": "SCREEN_PERCENT" },
  "inputs": {
    "startXPercent": { "source": "LITERAL", "value": 20 },
    "startYPercent": { "source": "LITERAL", "value": 20 },
    "endXPercent": { "source": "LITERAL", "value": 40 },
    "endYPercent": { "source": "LITERAL", "value": 30 }
  }
}
```

Add `FULL_SCREEN` to the accepted catalog mode names and declare it only
on Screenshot. The shared mode names keep separate binding schemas for
each node. Tap and Long Press retain selector targeting. Swipe retains its
two coordinate modes. Screenshot accepts no selector mode.

Require an explicit mode. Each mode requires exactly its own literal fields.
Check `startX < endX` and `startY < endY` in pixel mode, and
`startXPercent < endXPercent` and `startYPercent < endYPercent` in percentage
mode. An invalid mode produces `NODE_CONFIG_INVALID` at `config.targetMode`. Bad
bindings produce `NODE_BINDINGS_INVALID`. When each field is valid but an
axis is equal or reversed, point at the trailing edge's `value`. Backend
and local compilation preserve the mode, field names, and literal values.
G admission checks the same rules for supplied IR. F7 rechecks the resolved
values before ADB I/O. Apply the existing integral JSON number semantics,
so `20.0` is accepted as 20.

Before validating an existing editable Screenshot workflow against the
revised catalog, upgrade nodes with empty config and empty bindings to
explicit `FULL_SCREEN` and invalidate stale compiled IR. Previously frozen
definitions preserve their original full-screen meaning. The revised
definition requires an explicit mode for every invocation.

## 2. Capture one bounded frame

1. Check the attempt context, deadline, execution-owned serial, and resolved
   inputs. Invalid input stops before resolving or invoking ADB.
2. Use the `SCREENSHOT` operation from the [A4 typed device command
   contract](../contracts/typed_device_command_v1.md) with no workflow-provided
   arguments. F1 uses the E2-resolved executable, binds the exact serial,
   and runs the fixed argument vector through E1.

   ```text
   adb -s <serial> exec-out screencap -p
   ```

3. Capture stdout as binary PNG data with a **16 MiB** limit. Reuse E1's
   bounded stdout capture and truncation signal. Keep the ordinary command
   output and screenshot stderr on their existing diagnostic limits. Reject
   truncated stdout even when ADB exits zero. Empty stdout and unsuccessful
   process outcomes fail the attempt. Keep the same deadline through tool
   resolution, capture, image processing, and staging.
4. Read PNG dimensions before full decode. Require each dimension to be at
   most **4,096** and the frame to contain at most **12,000,000 pixels**.
   For a region, decode the bounded source image before cropping. For full
   screen, pass through the original bytes and use D5's full PNG validation
   before staging. D5 applies its own limits to the image it stores.

F7 never makes a separate `wm size` call. Dimensions and orientation come
from the PNG captured by this attempt, including for devices of different
sizes in one run group. The full frame stays in memory for a region capture.
The existing per-device execution and group concurrency limits bound how
many captures can be active. Release capture and decoded image references
after staging or failure. See [E1 process boundaries](../e_device_subsystem/e1_process_runner.md)
and [D5 PNG validation](../d_persistence/d5_artifact_store.md#3-validation-and-publication).

## 3. Resolve a pixel rectangle

In `ABSOLUTE_PIXELS`, use the four supplied integers as pixel edges. Check
`0 <= startX < endX <= width` and `0 <= startY < endY <= height` against the
captured frame. A right edge of `width` and a bottom edge of `height` are
valid. If the rectangle falls outside the frame, return a non-retryable
`COMMAND_INPUT_UNSUPPORTED` with a fixed message. Rotation or a smaller
device can cause this outcome. Runtime uses the exact region and performs
no clipping or scaling.

Treat percentages as image edges. The left and top edges are inclusive.
The right and bottom edges are exclusive. Round the leading edges down
and the trailing edges up so the crop includes the pixels it overlaps.
With integer division, the formulas are:

```text
x0 = (width  * startXPercent) / 100
y0 = (height * startYPercent) / 100
x1 = (width  * endXPercent + 99) / 100
y1 = (height * endYPercent + 99) / 100
```

Apply the formulas relative to the decoded image's top-left and require
the resolved rectangle to fit its bounds and have positive width and
height. Pixel index conversion used by Tap and Swipe is a separate rule.
Here, a trailing edge at 100% equals the image dimension. `(0, 0)` to
`(100, 100)` covers the whole frame.

| Captured frame | Selected mode and points | Pixel bounds `(x0,y0)` to `(x1,y1)` | Stored PNG |
|---|---|---|---|
| 1920 × 1080 | Pixels `(384,216)` to `(768,324)` | `(384,216)` to `(768,324)` | 384 × 108 |
| 1920 × 1080 | Percent `(20,20)` to `(40,30)` | `(384,216)` to `(768,324)` | 384 × 108 |
| 1080 × 1920 | Percent `(20,20)` to `(40,30)` | `(216,384)` to `(432,576)` | 216 × 192 |
| 7 × 5 | Percent `(20,20)` to `(40,30)` | `(1,1)` to `(3,2)` | 2 × 1 |

Encode the rectangle as PNG with its own dimensions. Bound encoded bytes
to 16 MiB and check the attempt context before and after decode and encode,
before staging, and before returning a staged result. Leave the captured
source bytes out of the artifact directory, database, logs, and events.
Cropping reduces stored and downloaded bytes for typical regions. The
full ADB capture still must fit the source limits.

## 4. Stage, commit, and retrieve

Use a Screenshot staging callback supplied by G for the active attempt
through the internal F9 invocation. Its contract is
`StageScreenshot(ctx, pngBytes) → (artifactID, error)`. The callback calls
D5 with the runtime-owned attempt identity and retains the staged metadata
in that attempt's execution scope. Validate that the callback is present
before capture in the execution path. F7 uses the returned ID as provisional
handler output. This leaves existing handlers' return types intact.

G sends the stage record and the matching output ID to D4. D4 commits the
metadata, successful attempt output, and events in the same SQLite
transaction. G exposes the output to later nodes only after D4 confirms
the commit. D5 and D4 own disposal after failed or stale finishes and
read-back after uncertain commits. If cancellation occurs after staging,
G still owns the stage record and must settle or reconcile it. D5 also owns
the stored size and dimensions, which describe the crop in region mode.

The client uses authenticated `GET /api/v1/artifacts/{artifactId}` to open
the committed PNG. D5 resolves the ID under the Runtime-controlled storage
root. Studio uses the same route for all three modes. See the
[D5 publication rules](../d_persistence/d5_artifact_store.md#3-validation-and-publication)
and [A7 artifact retrieval](../contracts/local_api_v1.md#11-artifact-retrieval).

## 5. Failures

| Condition | Required outcome |
|---|---|
| Invalid corners, fields, or config | `COMMAND_INPUT_UNSUPPORTED`, non-retryable. No ADB or storage I/O. |
| Pixel rectangle falls outside the captured frame | `COMMAND_INPUT_UNSUPPORTED`, non-retryable. Capture completed. No file is staged. |
| Missing execution serial, deadline, or staging callback | `INTERNAL_ERROR`. Stop before capture. |
| Resolution or process failure | Preserve the existing F1 classification and retryability. F7 makes at most one capture per attempt. |
| Empty, oversized, invalid, or truncated PNG, invalid dimensions, or crop encoding failure | `COMMAND_FAILED`, non-retryable. Leave no artifact. |
| D5 stage rejects image content | `COMMAND_FAILED`, non-retryable. Leave no successful output. |
| D5 cannot stage the validated image | `PERSISTENCE_FAILED`. Publish no output or artifact reference. |
| D4 commit failure or uncertainty | Use D4's existing failure and read-back path with the D5 artifact evidence. No workflow success while the commit is unconfirmed. |
| Cancellation or timeout | Preserve the attempt outcome. G settles the attempt and any staged file using the D4/D5 lifecycle. |

Return safe operation and reason information. Logs and persisted errors
exclude raw PNG bytes, ADB stdout and stderr, local paths, and device serials.

## 6. Implementation and integration order

| Order | Owner | Deliverable |
|---|---|---|
| 1 | Backend, Runtime validation, Studio | Define the three modes in the Backend catalog and validator. Sync the installed Runtime definition and validation rules before publication. Extend the parity fixtures and have Studio render the per-node schemas, including the zero-field mode. |
| 2 | D5 and D4 | Implement PNG staging, metadata, recovery, and atomic attempt publication. Verify with synthetic PNGs before adding ADB as a dependency. |
| 3 | F1 and F7 | Add the bounded binary Screenshot path under `internal/platform/adb` and the F7 handler under `internal/handlers/screenshot`. Use a controlled executable and an injected staging callback to exercise full-screen and region capture. |
| 4 | G, F9, and A7 | Supply the attempt-scoped staging callback, pass metadata into the finish checkpoint, and enable artifact retrieval by ID. Register and advertise the shipped handler once these paths and native acceptance pass. |

Use the same graph and IR envelope versions. Before synchronizing the
revised catalog to a shared environment, set its minimum Runtime version
to the first compatible release and verify the Studio decoder accepts
`FULL_SCREEN`. Keep the stored definitions for earlier frozen workflow
revisions. Defer selector regions, fractional percentages, variable-driven
corners, display selection, and multi-region capture.

## 7. Acceptance

| Check | Expected evidence |
|---|---|
| Full screen and both valid region modes | Backend and Runtime agree. Full screen carries an explicit mode and zero inputs. Both regions keep all four literals in IR. |
| Missing, unknown, or cross-node mode, extra or partial fields, wrong source, and bad geometry | Validators reject the same cases. An invalid resolved call starts zero ADB processes. Existing Tap, Swipe, and Long Press reject `FULL_SCREEN`. |
| Controlled ADB capture | One capture uses the exact serial and fixed `exec-out` arguments. PNG bytes reach F7 unchanged. Nonzero exit, cancellation, and stdout truncation publish no result. |
| Crop geometry | A color-coded test PNG verifies pixel accuracy, matching equivalent regions, edge rounding on odd dimensions, full-image bounds, and rejection when pixel bounds exceed a smaller frame. |
| Capture and image limits | Exercise the encoded byte boundary, oversized source, malformed and truncated images, axis and pixel-count limits, and an encoded crop exceeding the store limit. |
| Persistence and recovery | Success opens the expected PNG by `artifactId`. File or checkpoint failure exposes no success reference. Uncertain commit and restart preserve the committed PNG and remove confirmed orphans. |
| Native and live acceptance | Verify binary ADB arguments and capture on Linux and Windows. On Waydroid and a physical Android device, retrieve captures from all three modes and compare visible content and dimensions. Rotate the device and repeat. Inspect stored region files to confirm only the cropped content was committed. |
