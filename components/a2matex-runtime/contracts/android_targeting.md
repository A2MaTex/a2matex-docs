# Android targeting contract

**Status:** Planned execution contract. Implementation belongs to Runtime
Phases E–G and Studio integration.

## Overview

The Backend compiler copies one explicit targeting mode and its literal fields
into authoritative Runtime IR; the Runtime local compiler applies the same
mapping for Studio Test Run. Studio supplies source data but does not implement
the mapping. The Runtime targeting component resolves those inputs against the
selected Android device and dispatches a fixed command. Verify the component
independently before integrating workflow admission and Studio Test Run.

```mermaid
flowchart LR
  S[Backend or Runtime compiler] --> I[IR mode and literal inputs]
  I --> R[Runtime targeting resolver]
  R -->|Percent or selector| A[Read selected device state]
  A --> R
  R -->|Valid pixel target| D[Dispatch typed ADB action]
  R -->|Invalid or unavailable target| E[Return failure]
```

## Compilation

Studio discovers available modes and their authoring fields from the Backend
Node Definition `editorHints.targeting.modes`; it does not maintain a separate
`TARGET_FIELDS` map. Backend authoritative compilation and Studio local
compilation use that definition. The Runtime keeps an independent local
validation contract because execution cannot depend on a live catalog request.
Cross-language parity fixtures must keep the Runtime rules aligned with the
published catalog.

Keep the node ID and handler identity. Copy the required
`config.targetMode`. Map each Editor Graph binding
`{"kind":"LITERAL","value":100}` to the same field in IR `inputs` as
`{"source":"LITERAL","value":100}`. Preserve the value type. Every targeting
node must carry exactly the fields for its selected mode. Missing modes,
extra fields, and non-literal binding sources fail admission.

| Node | Absolute pixels | Screen percentage | Selector ID | Additional input |
|---|---|---|---|---|
| Tap | `x`, `y` | `xPercent`, `yPercent` | `selector_id` | None |
| Swipe | `startX`, `startY`, `endX`, `endY` | `startXPercent`, `startYPercent`, `endXPercent`, `endYPercent` | Unsupported | `durationMs` |
| Long Press | `x`, `y` | `xPercent`, `yPercent` | `selector_id` | `durationMs` |

Mode names are `ABSOLUTE_PIXELS`, `SCREEN_PERCENT`, and `SELECTOR_ID`.
Pixel coordinates are integers from 0 to 100,000. Percentages are integers
from 0 to 100. Durations are integers from 1 to 60,000 ms and must fit within
the attempt timeout to finish. Selector IDs use the full Android resource ID.

## Resolution and dispatch

- Pixels go straight to dispatch. Percentage targeting reads the current
  default display size and rounds each axis as
  `round(percent × (dimension − 1) / 100)`. Swipe uses one size snapshot.
- Selector targeting reads the current hierarchy, requires exactly one
  resource-ID match, and dispatches to the center of its screen-clipped
  bounds. Missing, ambiguous, hidden, malformed, or empty bounds fail before
  dispatch. The ADB adapter uses a Runtime-generated temporary device path
  with bounded output and best-effort cleanup.
- Dispatch uses the selected serial and fixed arguments. Long Press uses an
  Android input swipe with identical endpoints and the hold duration. These
  gestures expose an empty workflow output. Adapter errors carry no raw ADB
  output or hierarchy contents.

## Integration gates

Runtime must wire this component to the device inventory, executable IR
admission, handler registry, workflow runner, and checkpoints. Studio owns
the compiler integration. Backend and Launcher keep capability gating tied
to the Runtime build verified on real devices. Catalog presence alone is
authoring availability. Real-device checks must cover current display and
hierarchy output on the supported Android versions before enabling Test Run.
