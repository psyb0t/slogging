# Public API and compatibility

Prefix: API. Every exported identifier in `slogconf`, `handlers`, `handlers/logring` and `handlers/loki` is public API, and other repositories compile against it.

- **API1:** `slogconf/compat_test.go` pins the call shapes downstream code uses (`AddSink`, `SetOutput`, `SetHandlers` and `Init`), the `*handlers.FanOutHandler` default, the slot behavior of those calls, and the exported environment variable names. Do not edit it to make a change compile; a change that breaks it is a breaking change.
- **API2:** A breaking change, including a changed default, a renamed environment variable or a broken invariant from the handler rules, is called out as breaking in `CHANGELOG.md` and gets a row in the migration tables in `docs/migrating.md`.
- **API3:** The default environment variable names (`LOG_LEVEL`, `LOG_FORMAT`, `LOG_ADD_SOURCE`, `SLOGGING_LOKI_URL`, `SLOGGING_LOKI_APPNAME`) and the zero-value behavior of every `Options` struct stay as they are unless the change is released as breaking.
- **API4:** A new configuration knob is a field on the package's `Options` or config struct, read from the environment directly (no struct-tag config loader; see the `readConfig` comment in `slogconf/log.go`).
