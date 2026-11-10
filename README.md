# slogging

[![Go Reference](https://pkg.go.dev/badge/github.com/psyb0t/slogging.svg)](https://pkg.go.dev/github.com/psyb0t/slogging)
[![CI](https://github.com/psyb0t/slogging/actions/workflows/pipeline.yml/badge.svg?branch=main)](https://github.com/psyb0t/slogging/actions/workflows/pipeline.yml)
[![coverage](https://raw.githubusercontent.com/psyb0t/slogging/badges/coverage.svg)](https://github.com/psyb0t/slogging/actions/workflows/pipeline.yml)
[![version](https://raw.githubusercontent.com/psyb0t/slogging/badges/version.svg)](https://github.com/psyb0t/slogging/tags)
[![license](https://raw.githubusercontent.com/psyb0t/slogging/badges/license.svg)](https://github.com/psyb0t/slogging/blob/main/LICENSE)
[![imported by](https://raw.githubusercontent.com/psyb0t/slogging/badges/importers.svg)](https://github.com/psyb0t/slogging/blob/badges/importers.md)

Everything for Go's stdlib `log/slog` in one place: configure it from the environment, and stack handlers onto it that ship logs to Loki or keep them searchable in memory.

Spiritual successor to [`logrus-configurator`](https://github.com/psyb0t/logrus-configurator), and the continuation of `slog-configurator`. Coming from there? Read [Migrating from slog-configurator](https://psyb0t.github.io/slogging/migrating/).

## Contents

- [Documentation](#documentation)
- [Quick start](#quick-start)
- [Layout](#layout)
- [Development](#development)
- [Sponsor](#sponsor)
- [License](#license)

## Documentation

The full documentation lives at **[psyb0t.github.io/slogging](https://psyb0t.github.io/slogging/)**.

| Page | Covers |
| --- | --- |
| [Configuring slog](https://psyb0t.github.io/slogging/slogconf/) | `LOG_LEVEL`, `LOG_FORMAT`, `LOG_ADD_SOURCE`, renaming them, and `AddSink` vs `SetOutput` vs `SetHandlers` |
| [Output and fan-out handlers](https://psyb0t.github.io/slogging/handlers/) | The stdout and stderr split, several writers per stream, a level you can change at runtime, teeing to many handlers |
| [Searchable ring](https://psyb0t.github.io/slogging/logring/) | The in-memory ring: limits, attribute search, paging, stats |
| [Loki handler](https://psyb0t.github.io/slogging/loki/) | Settings, labels, line format, and what happens when Loki is down |
| [Troubleshooting](https://psyb0t.github.io/slogging/troubleshooting/) | The startup panic, doubled lines, sinks that see nothing, missing Loki records |
| [Migrating from slog-configurator](https://psyb0t.github.io/slogging/migrating/) | Old import paths and the pre-1.7 handler API |
| [Development](https://psyb0t.github.io/slogging/development/) | Make targets, tests, CI and releases |

## Quick start

```bash
go get github.com/psyb0t/slogging
```

```go
package main

import (
	"log/slog"

	_ "github.com/psyb0t/slogging/slogconf" // configures slog at init time
)

func main() {
	slog.Info("this is an info message", "user", "psyb0t", "action", "testing")
	slog.Error("this is an error message", "error_code", "E001")
}
```

```bash
export LOG_LEVEL="debug"      # debug / info / warn / error
export LOG_FORMAT="json"      # json / text
export LOG_ADD_SOURCE="true"  # include source file/line/function
go run main.go
```

**stdout/stderr are split automatically.** Info and debug go to stdout, warnings and errors to stderr. Container log collectors capture both and tag them, so error noise stays separate from happy-path noise.

Need more than the blank import? Rename the variables with `slogconf.Init`, add a destination with `slogconf.AddSink`, or swap where you print with `slogconf.SetOutput`. Those are different calls on purpose, and [Configuring slog](https://psyb0t.github.io/slogging/slogconf/) explains why.

## Layout

```
slogconf/              configure slog from the environment
handlers/              Handler (the process's output) + FanOutHandler (tees to many)
handlers/logring/      bounded in-memory ring you can search
handlers/loki/         push records to Loki's HTTP API
```

Two kinds of thing live here, and the split is the whole design. `handlers` holds the **structural** pieces: the one that writes your output, and the one that tees a record to many. Its subpackages are **destinations**: a searchable ring, a Loki server. You configure one output and add as many destinations as you like.

Runtime dependencies are `ctxerrors` and nothing else. No config loader, no HTTP framework. Every handler talks to `log/slog` and the standard library.

## Development

```bash
make test           # all tests with -race
make test-coverage  # coverage gate (fails under 90%)
make lint           # go fix + golangci-lint
make lint-fix       # same, with --fix
make docs           # build the docs site locally with Docker
```

See `make help` for the full list and [Development](https://psyb0t.github.io/slogging/development/) for CI and releases.

## Sponsor

<a href="https://www.buymeacoffee.com/psyb0t"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-blue.png" alt="Buy Me a Coffee" height="60" width="217"></a>

## License

MIT. See [LICENSE](https://github.com/psyb0t/slogging/blob/main/LICENSE).

See [CHANGELOG.md](https://github.com/psyb0t/slogging/blob/main/CHANGELOG.md) for release notes.
