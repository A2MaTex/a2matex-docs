# Typed Device Command v1 Contract

**Status:** Accepted

**Phase:** A4 — Typed device command contract

## 1. Purpose

A registered Android node handler converts validated node inputs into one typed
device command. The dispatcher binds that command to the `adbSerial` owned by
the current `DeviceExecution`, and the ADB adapter converts it into fixed
executable arguments.

This boundary prevents workflow-controlled values from becoming executable
names, shell syntax, raw ADB arguments, target serials, or filesystem paths.
Typed commands are internal Runtime values, not a public JSON API and not a
publisher-extensible command language.

```text
validated node inputs
        -> registered handler
        -> typed device command
        -> dispatcher + DeviceExecution.adbSerial
        -> ADB adapter arguments
        -> process result
        -> command outcome or A6 error
```

## 2. Command variants

The MVP command set is closed. Exactly these variants are supported:

| Command | Fields | Source nodes |
|---|---|---|
| `LAUNCH_APP` | `packageName` | `android.launch_app` |
| `TAP` | `x`, `y` | `android.tap` |
| `SWIPE` | `startX`, `startY`, `endX`, `endY`, `durationMs` | `android.swipe` |
| `LONG_PRESS` | `x`, `y`, `durationMs` | `android.long_press` |
| `INPUT_TEXT` | `text` | `android.input_text` |
| `PRESS_BACK` | None | `android.press_back` |
| `PRESS_HOME` | None | `android.press_home` |
| `SCREENSHOT` | None | `android.screenshot` |

Flow and utility nodes do not produce device commands. Unknown command variants
and fields are rejected; the dispatcher never falls back to raw execution.

## 3. Field rules

A handler receives values only after Runtime IR binding resolution and node
input validation. A4 preserves the approved Backend Node Catalog constraints:

| Field | Rule |
|---|---|
| `packageName` | String of 3–255 characters matching the approved Android package-name pattern |
| Coordinates | Integers from 0 through 100,000 |
| Swipe and Long Press `durationMs` | Integer from 1 through 60,000 |
| `text` | String accepted by the `android.input_text` node contract, with a maximum length of 10,000 |

The command contract does not perform implicit type conversion, truncation,
coordinate scaling, device substitution, or fallback-value resolution. If the
adapter cannot represent accepted input text without changing its meaning, it
returns an A6-classified failure instead of executing an altered command.

`SCREENSHOT` has no fields. In particular, workflow data cannot select its
destination path, filename, format, or overwrite behavior.

## 4. Target and execution ownership

The typed command contains none of the following:

- ADB serial or persistent device identity.
- Executable or ADB binary path.
- Raw argument list or shell command.
- Timeout, retry count, or error policy.
- Run, device-execution, node, or attempt identifiers.
- Artifact path or arbitrary local path.

These values have separate owners:

| Concern | Owner |
|---|---|
| Target serial | Current A5 `DeviceExecution.adbSerial` |
| ADB executable | A1 resolution policy and Phase E toolchain resolver |
| Fixed ADB arguments | Phase F dispatcher and adapter implementation |
| Timeout and cancellation | Node-attempt context created by Phase G |
| Retry and error transition | Phase G workflow engine |
| Correlation identifiers and persistence | A5 execution contracts |
| Error code and safe details | A6 error contract |
| Artifact path and metadata | Runtime artifact store |

The dispatcher accepts the command and target serial as separate trusted
arguments. It always invokes the selected ADB executable directly with a
separate argument array; it never joins workflow values into a shell string.

## 5. Command outcomes

For Launch App, Tap, Swipe, Long Press, Input Text, Back, and Home, successful dispatch has
no command payload and produces no workflow-visible node output. Failure
returns an A6 error.

For Screenshot, the adapter returns captured PNG content to Runtime-controlled
artifact handling. After validation and storage, the handler returns only the
`artifactId` required by the node output contract. Width, height, media type,
checksum, path, and size belong to artifact metadata. Raw image bytes are not
embedded in execution events or node output JSON.

Exit code, bounded stdout/stderr, and measured process duration belong to the
safe process runner. Node-attempt timing belongs to A5 execution records. They
are not typed-command fields or workflow-visible node outputs.

A successful command means only that the operation completed at the ADB
boundary. It does not prove an observed UI state or a business result. A
non-zero exit, timeout, cancellation, invalid adapter response, or unsafe input
representation returns an A6 error instead of a successful outcome.

## 6. Safety invariants

- Only registered system handlers can create commands for dispatch.
- Workflow data cannot provide a raw command or select an executable.
- The command cannot select or replace its target device.
- Every dispatch uses the serial from the current device execution.
- The adapter builds arguments for the recognized command variant only.
- Workflow text is never evaluated by a general shell.
- Unknown variants fail closed.
- Process diagnostics are bounded and sanitized by their owning components
  before logging, persistence, events, or user-visible errors.
- Command completion never implies an observed UI state or confirmed business
  result.

## 7. Excluded responsibilities

A4 does not define:

- Device discovery, selection, or readiness.
- ADB process execution code.
- Execution persistence or lifecycle states.
- Error codes and public error envelopes.
- Workflow retry, timeout, or transition behavior.
- Backend node input/output schemas.
- Observation, OCR, selector, or business-verification nodes.

## 8. Checkout condition

A4 is complete when all eight current Android nodes map to one of eight closed
command variants, all fields preserve the approved node constraints, and no
workflow-controlled value can choose an executable, raw argument sequence,
target serial, shell syntax, or artifact path.

Phase F will implement contract-focused Go tests, real ADB argument
construction, and real-device acceptance tests. Cross-language fixtures are not
needed because Studio and Backend never create or dispatch these internal
commands.
