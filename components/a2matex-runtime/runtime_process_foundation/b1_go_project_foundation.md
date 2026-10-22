# B1 — Go Project Foundation

**Status:** Proposed

**Phase:** B1 — Go project skeleton

## 1. Responsibility

B1 reconciles the existing Go scaffold with the accepted Runtime plan before
feature implementation continues. It establishes the module identity, one
runnable agent binary, private package boundary, compile-time dependency
wiring, build metadata, developer commands, and test placement convention.

The scaffold predates A1–A8. Existing files are useful starting points, but do
not make B2 configuration, B3 supervision, B4 logging, B5 authentication, or B6
Local API behavior complete.

## 2. Foundation decisions

| Area | B1 decision |
|---|---|
| Application | One long-running local Runtime Agent process |
| Binary | Keep `cmd/agent`; build it as `a2matex-runtime` |
| Packages | Keep implementation under `internal`; do not add `pkg` |
| Module | Use `github.com/A2MaTex/a2matex-runtime` |
| Dependency injection | Constructor injection with Google Wire only in `cmd/agent` |
| Existing libraries | Retain Gin, Viper, Wire, and standard-library `log/slog` |
| Go version | Retain the current Go 1.25 module target |
| Generated source | Commit `wire_gen.go`; never edit it manually |
| Development version | Use valid SemVer `0.0.0-dev` |
| Tests | Colocate tests with their source packages |

No additional binary is needed now. A future worker may add `cmd/<name>` when
it has approved behavior, but B1 must not create empty entrypoints.

## 3. Target shape

```text
a2matex-runtime/
├── cmd/agent/                 # process entrypoint and Wire composition
├── configs/                   # tracked example; local config is ignored
├── internal/
│   ├── app/                   # application orchestration
│   ├── buildinfo/             # linker-provided build identity
│   ├── config/                # B2 ownership
│   ├── logging/               # B4 ownership
│   └── transport/http/        # B6 ownership
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

This is an ownership map, not a requirement to create placeholder packages.
Later phases add packages only when they implement approved behavior.

## 4. Required reconciliation

### Project and composition

- Replace the placeholder module name and update all internal imports.
- Keep `main.go` limited to process input, root context, construction, running
  the application, and mapping the final result to an exit code.
- Keep `wire.go` as the only injector definition. Packages use explicit
  constructors and never receive a container or service locator.
- Regenerate `wire_gen.go` after provider or import changes.
- Keep one Go module; do not introduce `go.work`.

Wire constructs dependencies only. B3 owns component lifecycle and graceful
shutdown.

### Build and developer workflow

- Keep version, commit, and build time linker-injected.
- Default the non-release version to `0.0.0-dev`.
- Keep commit and build time internal; B6 may expose only A7's
  `runtimeVersion`.
- Provide `wire`, `fmt`, `test`, `build`, `lint`, and `run` Make targets.
- Keep generated and build outputs ignored.
- Correct README paths to `configs/` and distinguish implemented behavior from
  planned B2–B6 behavior.

A release pipeline later supplies real build values. Installer, packaging, and
auto-update design remain deployment work.

## 5. Implementation sequence

1. Update the module path and imports.
2. Set the valid development version.
3. Add the missing Make target and confirm ignore rules.
4. Regenerate Wire output from `wire.go`.
5. Correct README paths, commands, and implementation-status wording.
6. Run verification and inspect the final diff for behavior changes.

## 6. Explicit exclusions

B1 does not:

- add dependencies, an ORM, a database driver, or another DI framework;
- finalize configuration fields or filesystem paths;
- implement a lifecycle framework, authentication, or A7 responses;
- add workflow, device, persistence, event, or node-handler behavior;
- add Production mode behavior; or
- design release packaging or CI automation.

The preliminary config, logger, application, and HTTP server may remain
buildable. Their behavioral reconciliation belongs to B2–B6.

## 7. Verification and test proposal

```text
make wire
make fmt
make test
make build
make lint
git diff --check
```

B1 changes metadata and wiring rather than product behavior. No new test
dependency is required. Any new unit tests remain subject to approval before
B1 implementation.

## 8. Checkout condition

B1 is complete only when:

- the module path matches the repository;
- one `a2matex-runtime` binary builds from `cmd/agent`;
- Wire generation is reproducible and the composition root remains small;
- the default Runtime version is valid SemVer;
- Make commands and README paths match the checkout;
- source tests, build, formatting, lint, and diff checks pass; and
- no B2–B6 behavior is claimed complete.
