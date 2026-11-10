# Go code

Prefix: GO.

- **GO1:** Test files are excluded from linting by `.golangci.yml`, not from review.
- **GO2:** Imports are limited by `depguard` to the standard library, `github.com/psyb0t/...` and `github.com/stretchr/testify`. The only runtime dependency is `github.com/psyb0t/ctxerrors`, and the README and docs say so.
- **GO3:** A caller-checkable failure gets an exported sentinel in the package's `errors.go` (`ErrInvalidLogLevel`, `ErrInvalidFormat`), wrapped so `errors.Is` matches it.
- **GO4:** The one `init` in `slogconf/log.go` is the module's reason to exist and carries `//nolint:gochecknoinits`; do not add another.
- **GO5:** Environment variable names are exported constants (`slogconf.EnvVarNameLevel`, `loki.EnvVarNameURL`) so callers and tests use the same name.
