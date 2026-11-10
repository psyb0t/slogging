# Testing

Prefix: TEST. How to run the suites is in `AGENTS.md` and `docs/development.md`.

- **TEST1:** Tests sit beside the code. Internal tests use the package name and the `_internal_test.go` suffix. External tests (`package <name>_test`) are for checks from a caller's point of view: `slogconf/compat_test.go` and `handlers/logring/integration_test.go`.
- **TEST2:** Use testify (`require` for preconditions, `assert` for checks). Use `t.Setenv` for environment variables and `httptest.NewServer` for Loki (`handlers/loki/loki_internal_test.go`). Tests never reach the network.
- **TEST3:** A test that calls `slogconf.Init`, `SetHandlers`, `SetOutput`, `AddSink` or `slog.SetDefault` changes process-global state. Restore it in `t.Cleanup` (the integration test calls `slogconf.Init(slogconf.Options{})` there) and do not mark such a test `t.Parallel()`.
- **TEST4:** Tests run with `-race`. Concurrency claims in comments (the ring's locks, shared `fmtMu`) need a test that exercises them under the race detector.
- **TEST5:** The coverage floor is 90 percent (`MIN_TEST_COVERAGE` in the `Makefile`). Cover the code you add instead of lowering the floor.
- **TEST6:** A bug fix starts with a test that fails on the old behavior.
