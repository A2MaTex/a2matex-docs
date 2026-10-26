# Phase B — Runtime Process Foundation

Phase B implements the Go process shell required by every later Runtime
subsystem. It turns the accepted Phase A contracts into a runnable,
authenticated, loopback-only agent foundation, but it does not execute a
workflow or communicate with an Android device.

## Progress

| ID | Module | Status |
|---|---|---|
| B1 | Go project foundation | Implemented and verified; lint pending |
| B2 | Configuration | Implemented and verified; lint pending |
| B3 | Supervisor | Implemented and focused verification passed; lint pending |
| B4 | Logging and redaction | Implemented; focused logger test skipped by review decision; lint pending |
| B5 | Local authentication | Implemented; Windows verification and lint pending |
| B6 | Minimal Local API | Implemented and verified; lint pending |

Each sub-phase is specified, approved, implemented, and verified separately.
Documentation approval does not mark its Go implementation complete.

## B1 — Go Project Foundation

**Status:** Implemented and verified; lint pending

B1 reconciles the existing scaffold without adding Runtime features. It keeps
one `cmd/agent` binary, private `internal` packages, explicit constructors, and
Google Wire at the composition root. It corrects project identity, build
metadata, generation workflow, developer commands, and documentation before
later foundation behavior is changed.

Reference:

- [B1 — Go Project Foundation](../runtime_process_foundation/b1_go_project_foundation.md)

## B2 — Configuration

**Status:** Implemented and verified; lint pending

B2 defines deterministic startup configuration for the components implemented
in B3–B6. It keeps only shutdown, loopback HTTP, and logging settings; applies
strict precedence and validation; and defers every setting without an
implemented consumer. It exposes no mode setting or fixed mode response field;
Production behavior remains I2 scope.

Reference:

- [B2 — Configuration](../runtime_process_foundation/b2_configuration.md)

## B3 — Runtime Supervisor

**Status:** Implemented and focused verification passed; lint pending

B3 makes `internal/supervisor.App` the sole process-lifecycle owner. It defines
synchronous readiness, ordered startup, startup rollback, unexpected-failure
propagation, reverse shutdown, and one B2-bounded cleanup deadline. The current
implementation manages the preliminary Local HTTP Server and the B5 connection
file publisher. Later phases add components only when their behavior exists.
This matters because components own different resources or background
activities: each implements its own preparation and cleanup, while the
supervisor guarantees safe process-wide ordering and completion.

Reference:

- [B3 — Runtime Supervisor](../runtime_process_foundation/b3_supervisor.md)

## B4 — Logging and Redaction

**Status:** Implemented; focused logger test skipped by review decision; lint pending

B4 makes the existing `log/slog` path a safe operational boundary. It defines
stable lifecycle and Local API fields, central key-based redaction, safe panic
reporting, bounded request records, and reserved execution-correlation names.
It adds no logging dependency or unused execution infrastructure.

Reference:

- [B4 — Logging and Redaction](../runtime_process_foundation/b4_logging_and_redaction.md)

## B5 — Local Authentication

**Status:** Implemented; Windows verification and lint pending

B5 creates one cryptographically random local session per Runtime start,
verifies strict bearer credentials in constant time, and publishes the complete
A7 base URL and token document through a protected per-user connection file.
The publisher starts after the loopback listener and is removed before HTTP
shutdown. B6 remains responsible for applying the verifier to routes and
returning the exact A7 unauthorized response.

Reference:

- [B5 — Local Authentication](../runtime_process_foundation/b5_local_authentication.md)

## B6 — Minimal Local API

**Status:** Implemented and verified; lint pending

B6 connects the B5 session verifier to the existing HTTP server. Exact health
remains public, while every other request requires the per-start bearer token.
It replaces the preliminary handlers with the accepted A7 health and
Runtime-information shapes without adding future validation, device, execution,
or node-handler infrastructure.

Reference:

- [B6 — Minimal Local API](../runtime_process_foundation/b6_minimal_local_api.md)

Database, artifact, ADB, SSE, and execution-concurrency settings stay with
their later owning phases instead of becoming unused Phase B fields.

## Phase B gate

Phase B is complete when the agent starts, publishes protected connection
information only after the loopback listener is ready, serves the exact B6 API,
rejects unauthenticated requests, and exits within its shutdown deadline
without leaking owned goroutines or credentials.
