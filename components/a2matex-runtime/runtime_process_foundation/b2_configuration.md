# B2 — Configuration

**Status:** Implemented and verified; lint pending

**Phase:** B2 — Runtime process foundation

## 1. Responsibility

B2 gives the Runtime one deterministic, validated startup configuration. It
owns configuration loading, precedence, normalization, and validation for the
foundation components implemented in B3–B6. Invalid configuration stops the
process before any listener, worker, or connection file is created.

B2 does not collect settings for unimplemented subsystems. ADB, persistence,
artifacts, event retention, and execution concurrency add their settings in
their owning phases when those settings have consumers.

## 2. Configuration shape

```yaml
shutdown_timeout: 10s

http:
  listen_address: 127.0.0.1:8765
  read_header_timeout: 5s
  read_timeout: 10s
  write_timeout: 30s
  idle_timeout: 60s

logging:
  level: info
  format: json
```

All fields have built-in defaults, so a configuration file is optional.
`configs/config.example.yaml` is a documented template, not an automatically
loaded runtime file.

## 3. Field contract

| Key | Default | Accepted value | Consumer |
|---|---:|---|---|
| `shutdown_timeout` | `10s` | Go duration from `1s` through `1m` | B3 supervisor |
| `http.listen_address` | `127.0.0.1:8765` | Numeric loopback IP and non-zero port | B6 HTTP server |
| `http.read_header_timeout` | `5s` | Go duration from `1s` through `30s` | B6 HTTP server |
| `http.read_timeout` | `10s` | Go duration from `1s` through `2m` | B6 ordinary requests |
| `http.write_timeout` | `30s` | Go duration from `1s` through `2m` | B6 ordinary responses |
| `http.idle_timeout` | `60s` | Go duration from `1s` through `5m` | B6 keep-alive connections |
| `logging.level` | `info` | `debug`, `info`, `warn`, or `error` | B4 logger |
| `logging.format` | `json` | `json` or `text` | B4 logger |

Duration values use Go duration syntax, such as `500ms`, `10s`, or `1m30s`.
Bare numbers are invalid. Enum values are normalized to lowercase.

The listen address must parse as an IP and port. IPv4 and IPv6 loopback
addresses are accepted; hostnames, wildcard addresses, LAN addresses, missing
ports, and port `0` are rejected. This preserves the A1 loopback-only boundary
and avoids an undiscoverable randomly assigned port.

`http.write_timeout` is the ordinary-response budget. It must not become a
global deadline that terminates the A7 SSE stream. The later SSE transport uses
heartbeat-aware write deadlines as required by A7.

There is no `mode` configuration or fixed mode response field. The current API
is Development-only. I2 may introduce a mode only when Production behavior has
an approved contract and implementation.

## 4. Sources and precedence

The resolved value order is:

1. supported command-line overrides;
2. `A2MATEX_RUNTIME_*` environment variables;
3. an explicitly supplied YAML file;
4. built-in defaults.

The supported flags remain:

```text
-config <path>
-listen-address <ip:port>
-log-level <level>
```

When `-config` is absent, Runtime does not search the working directory and
does not load a file. When it is present, a missing, unreadable, malformed, or
invalid file is fatal. Relative paths are resolved from the process working
directory.

Environment variables replace nested dots with underscores, for example:

```text
A2MATEX_RUNTIME_SHUTDOWN_TIMEOUT=15s
A2MATEX_RUNTIME_HTTP_LISTEN_ADDRESS=127.0.0.1:9000
A2MATEX_RUNTIME_LOGGING_LEVEL=debug
```

An empty environment value is treated as unset. Configuration is loaded once
at startup; B2 does not implement file watching or live reload.

## 5. Validation and safety

- Decode into typed structs with explicit `mapstructure` tags.
- Reject unknown YAML keys rather than silently accepting a typo.
- Validate the fully merged configuration, not each source independently.
- Return field-specific errors without printing the whole configuration.
- Never accept the B5 session token or another secret from configuration.
- Never log configuration wholesale.
- Pass the validated configuration by value through constructor injection;
  packages do not access Viper directly.

Before B4 is available, startup configuration failures are written once to
standard error and return a non-zero exit code. B2 creates no goroutine and
opens no long-lived resource.

## 6. Deferred settings

| Setting | Owning phase |
|---|---|
| ADB executable override and resolution | E2 |
| Database engine, file, migrations, and retention | D1–D3 |
| Artifact root and limits | D5 |
| Device execution concurrency | H2 |
| Production package and authorization settings | I2 |
| Connection-file path and session token | B5 |

Workflow content can never provide any of these process-level settings.

## 7. Implementation and verification

Implementation:

1. remove the unused `mode` field and default;
2. stop implicitly loading `configs/config.yaml`;
3. retain the existing Viper dependency and standard `flag` package;
4. make decoding strict and add the exact bounds above;
5. update the example configuration and README; and
6. add focused table-driven configuration tests.

The tests should cover defaults, every precedence layer, nested environment
keys, strict unknown-key rejection, explicit-file failures, all enum values,
duration boundaries, and accepted/rejected listener addresses.

The implemented tests cover only B2 configuration behavior; they do not test
B3–B6 behavior.

Verification commands:

```text
make fmt
make test
make build
make lint
git diff --check
```

## 8. Checkout condition

B2 is complete when configuration is deterministic without a file, precedence
is proven by tests, invalid or unknown values fail before startup side effects,
only loopback listeners are accepted, no unimplemented subsystem has a
placeholder setting, and B3–B6 can consume typed configuration without
depending on Viper.
