# B4 — Logging and Redaction

**Status:** Implemented; focused logger test skipped by review decision; lint pending

**Phase:** B4 — Runtime process foundation

## Responsibility

B4 turns the Runtime's existing `log/slog` setup into a safe operational
logging boundary. It defines stable records for the process lifecycle and
Local API, prevents known sensitive attributes from reaching either configured
output format, and reserves correlation field names for later execution
phases.

B4 does not introduce logging as a new subsystem. B1 already constructs and
injects a `*slog.Logger`, B2 already validates `logging.level` and
`logging.format`, and B3 already emits preliminary lifecycle records. B4 keeps
those foundations and makes their behavior explicit and safe before B5 adds a
local session token.

## Why B4 matters

The current logger can produce JSON or text and filter records by level, but it
does not yet define which attributes are safe. In particular, the preliminary
panic middleware logs the recovered value directly, future Local API paths can
contain identifiers, and a later caller could attach a token, workflow
payload, or raw command output to any `slog` record.

B4 establishes two complementary protections:

1. call sites emit only approved stable messages and safe attributes; and
2. the common handler redacts known forbidden attribute keys as a final safety
   net.

Key-based redaction cannot discover a secret embedded in an arbitrary message
or error string. Safe construction at the call site remains mandatory.

## Existing foundation retained

B4 retains the current design:

- standard-library `log/slog`;
- synchronous output to standard output after configuration succeeds;
- JSON as the default format and text as the local-development alternative;
- B2 `debug`, `info`, `warn`, and `error` level selection;
- constructor injection into `internal/supervisor` and `internal/api`;
- one process-default logger for the final `cmd/agent` boundary; and
- startup configuration errors written once to standard error before the
  configured logger exists.

B4 adds no dependency, goroutine, queue, buffer, file sink, remote sink,
rotation mechanism, or lifecycle component.

## Operational logs are not execution events

B4 logs diagnose Runtime process behavior. They are not authoritative state
and are never used to reconstruct a run.

| Concern | Owner |
|---|---|
| Process lifecycle, request completion, internal diagnostics | B4 logs |
| Persisted RunGroup, DeviceExecution, and NodeAttempt state | A5/D phases |
| Replayable committed state changes | A6 events |
| SSE framing and recovery | A7/D phases |
| Publisher-authored `utility.log` content | Deferred until its sensitive-data contract is approved before F8 |

The Runtime must persist authoritative execution state before publishing an A6
event. It does not need to persist a B4 log record before continuing.

## Record shape

The selected standard-library handler continues to own the built-in `time`,
`level`, and `msg` attributes. B4 attributes use lowercase `snake_case`.

Every Runtime-owned operational record contains:

| Field | Type | Rule |
|---|---|---|
| `component` | String | Stable bounded subsystem name |
| `event` | String | Stable bounded event name, independent of human message text |

Optional fields are added only when the event owns them:

| Field | Type | Rule |
|---|---|---|
| `method` | String | HTTP method from the parsed request |
| `route` | String | Registered Gin route pattern, never the raw URL |
| `status` | Integer | HTTP response status |
| `duration_ms` | Integer | Non-negative elapsed milliseconds |
| `reason` | String | One of the bounded values defined by the emitting event |
| `error` | Error or string | Trusted internal error only; see error rules below |

Messages remain short, stable, and written by Runtime code. Dynamic identifiers
and values belong in attributes, not in `msg`.

## Current lifecycle records

B4 standardizes the lifecycle records already emitted by B3:

| Level | `component` | `event` | Message | Additional fields |
|---|---|---|---|---|
| Info | `runtime` | `runtime_starting` | `starting runtime agent` | None |
| Info | `runtime` | `runtime_ready` | `runtime agent ready` | None |
| Info | `runtime` | `runtime_stopped` | `runtime agent stopped` | None |
| Error | `runtime` | `runtime_failed` | `runtime agent stopped with an error` | Safe `error` |

`runtime_ready` means every currently registered B3 component completed
`Start`. In the present checkout that means only that the loopback listener is
bound. It does not claim A7 compliance, authentication, persistence, device
readiness, or execution readiness.

The B3 component name used in errors must be a stable non-secret identifier.
Current B4 uses `local_http_server`; later components must follow the same
lowercase `snake_case` convention.

Internal packages return errors with context instead of both logging and
returning them. `cmd/agent` logs the final non-nil application error once.

## Local API request logs

For a request that completes without a panic, the request middleware emits one
completion record with:

| Field | Value |
|---|---|
| `component` | `local_http_server` |
| `event` | `local_api_request_completed` |
| `method` | Request method |
| `route` | `gin.Context.FullPath()` after routing |
| `status` | Response status |
| `duration_ms` | Monotonic elapsed duration rounded down to milliseconds |

If Gin has no registered route, `route` is the fixed value `unmatched`. The
middleware never falls back to `Request.URL.Path`.

Request log levels are:

| Condition | Level |
|---|---|
| Successful `GET /api/v1/health` | Debug |
| Other status below 400 | Info |
| Status 400 or greater | Error |

The request record must not include:

- raw URL or query string;
- path parameters copied from the URL;
- headers, including `Authorization` and cookies;
- request or response bodies;
- client IP, forwarded address, or user agent; or
- the B5 session token or connection-file content.

This policy keeps request records bounded and prevents identifiers in future
run routes from changing the stable route classification.

## Panic recovery

The Local API recovery boundary returns the approved generic internal-error
response and emits exactly one fixed diagnostic:

| Field | Value |
|---|---|
| Level | Error |
| `component` | `local_http_server` |
| `event` | `local_api_panic_recovered` |
| Message | `recovered local API panic` |

The record contains no recovered value, stack trace, request data, raw path, or
technical response detail. The panic record and generic HTTP response must not
reveal the cause to the client. A recovered panic emits this diagnostic instead
of the ordinary request-completion record, so the failure is logged once.

## Redaction boundary

The selected JSON or text handler uses one common standard-library
`slog.HandlerOptions.ReplaceAttr` function. It inspects every non-group
attribute, including attributes inside nested groups, before serialization.

Forbidden keys are matched case-insensitively using the canonical key and the
common camel-case alias where applicable. Their values are replaced by the
fixed string `[REDACTED]`.

The initial forbidden set covers:

| Category | Forbidden keys or aliases |
|---|---|
| Authentication | `authorization`, `cookie`, `set_cookie`, `session_token`, `sessionToken`, `token` |
| Secrets | `secret`, `password`, `credential`, `credentials` |
| HTTP content | `request_body`, `requestBody`, `response_body`, `responseBody`, `payload` |
| Workflow content | `runtime_ir`, `runtimeIR`, `workflow_input`, `workflowInput`, `workflow_output`, `workflowOutput` |
| Device/process content | `adb_serial`, `adbSerial`, `stdout`, `stderr` |
| Local storage | `artifact_path`, `artifactPath`, `file_path`, `filePath` |

The replacement is deliberately key-based. B4 does not scan values with
regular expressions, attempt token-shaped value detection, or serialize an
object to inspect its contents. Those techniques add cost and still cannot
reliably identify arbitrary protected data.

Redaction is a safety net, not permission to attach sensitive values under a
different key. Call sites must still obey the prohibited-data rules.

## Prohibited data

No B4 operational record may contain:

- local session tokens, authorization headers, cookies, credentials, secrets,
  or Production authorization material;
- Runtime connection-file contents;
- Editor Graphs, protected Runtime IR, decrypted packages, workflow inputs,
  node configurations, bindings, or arbitrary node outputs;
- publisher-authored `utility.log` messages;
- ADB serials, unrelated device metadata, or persistent device fingerprints;
- raw process stdout/stderr, external-service responses, or raw ADB arguments;
- request/response bodies or uncontrolled values copied into the message;
- artifact bytes, arbitrary local filenames, or uncontrolled paths; or
- raw recovered panic values or stack traces.

An internal error may be logged only when its complete string and unwrap chain
are safe by construction. Code handling untrusted or protected values must
translate the failure into a stable internal classification before returning
it to a logging boundary.

## Correlation reserved for later consumers

B4 reserves these field names so later execution components do not invent
incompatible variants:

| Field | Meaning | First expected owner |
|---|---|---|
| `run_group_id` | A5 RunGroup UUID | H1 |
| `device_execution_id` | A5 DeviceExecution UUID | H1/H3 |
| `node_attempt_id` | A5 NodeAttempt UUID | H3 |
| `node_id` | Runtime IR node identifier | H3 |

These identifiers may correlate future operational records because they are
already controlled Runtime identifiers. `device_execution_id`, not
`adb_serial`, is the approved device correlation.

B4 does not add a context carrier, correlation struct, helper API, or unused
attributes before a real consumer exists. The owning later phase adds the
smallest helper needed while preserving these names and the B4 redaction
boundary.

## Level policy

| Level | Use |
|---|---|
| Debug | Expected high-frequency diagnostics, including successful health probes |
| Info | Normal lifecycle transitions and ordinary successful API operations |
| Warn | Explicit caller-selected degraded conditions; never inferred from an HTTP status |
| Error | All unsuccessful HTTP responses, process/component failures, and recovered panics requiring attention |

The same condition uses the same level at every call site. Sensitive data must
not be added at Debug under the assumption that production uses Info.

## Performance and bounds

B4 preserves synchronous standard-output logging. For each enabled record, the
additional work is bounded by the number and length of its attributes:

```text
emit record
→ inspect each attribute key once
→ replace forbidden values
→ serialize through the selected standard handler
→ write to stdout
```

No payload parsing, value regular expression, stack capture, network call, or
secondary sink occurs. Successful health probes are Debug to avoid dominating
normal logs. Sampling, asynchronous queues, and buffering are unnecessary for
the loopback MVP and remain deferred until measured volume proves a need.

## Implementation shape

The implementation changes only the current logging call path:

1. `internal/logging/logger.go` adds the shared `ReplaceAttr` redactor and
   retains the existing level/format handlers.
2. `internal/supervisor/supervisor.go` adds stable lifecycle attributes and uses the
   log-safe component identifier.
3. `internal/api/middleware` logs route patterns, applies the level policy,
   and removes raw recovered panic values.
4. `cmd/agent/main.go` adds the stable failure attributes while remaining the
   single terminal error logging boundary.
5. Focused tests cover logger output and HTTP log behavior.

No Wire constructor changes or generated-file edits are expected. If a
constructor signature changes unexpectedly, update `cmd/agent/wire.go` and
regenerate `wire_gen.go`; never edit generated output manually.

## Required verification

Focused standard-library tests must prove:

- JSON and text handlers retain the configured minimum level;
- safe attributes pass through unchanged;
- every initial forbidden key is redacted in both formats;
- nested group attributes are redacted;
- messages and stable fields remain unchanged;
- successful health requests are Debug;
- ordinary requests use Info and every client/server error request uses Error;
- request logs contain the route pattern and never raw path, query, headers,
  or body content;
- unmatched routes use the fixed `unmatched` value; and
- panic recovery logs no recovered value, stack, or request data.

The reviewer explicitly skipped a dedicated `logger_test.go`. Existing Local
API tests continue to cover secret-free request logging and panic recovery;
the remaining handler-level redaction cases are accepted by source review for
Phase B and can be added later without changing Runtime behavior.

Repository verification remains:

```text
make fmt
make test
make build
git diff --check
```

Lint remains separately reported when `golangci-lint` is unavailable or cannot
run in the environment.

## Deferred work

B4 intentionally does not add:

- persisted logs, log download, or log streaming APIs;
- A6 event payloads or SSE messages;
- publisher-authored `utility.log` behavior;
- execution correlation helpers without consumers;
- ADB command logging or raw process output;
- request IDs, traces, metrics, profiling, or OpenTelemetry;
- sampling, asynchronous logging, multi-sink fan-out, or remote collectors;
- file output, rotation, retention, or upload; or
- live logging reconfiguration.

Each deferred capability requires a concrete consumer, data-safety boundary,
and focused owning phase.

## Checkout condition

B4 is complete when current process and Local API logs are structured with
stable fields, known forbidden attributes are redacted in both configured
formats, request logs expose no uncontrolled request data, panic recovery
emits no recovered value, lifecycle failures are logged once at the process
boundary, the focused tests pass, and no unused execution or external logging
infrastructure has been introduced.
