---
title: "Home"
description: "slogging configures Go's log/slog from LOG_LEVEL, LOG_FORMAT and LOG_ADD_SOURCE, and adds handlers that push to Loki or keep logs searchable in memory."
nav_order: 1
permalink: /
seo:
  type: WebSite
---

# slogging

Everything for Go's stdlib `log/slog` in one place. One blank import configures the default logger from the environment, and the handler packages stack extra destinations onto it: a Loki shipper and a bounded in-memory ring you can search from inside the process.

It is the continuation of `slog-configurator` and the spiritual successor to [`logrus-configurator`](https://github.com/psyb0t/logrus-configurator). Coming from the old module path? Read [Migrating from slog-configurator](migrating.md).

## Install

```bash
go get github.com/psyb0t/slogging
```

The module declares `go 1.26` in its [`go.mod`](https://github.com/psyb0t/slogging/blob/main/go.mod). Its only runtime dependency is [`ctxerrors`](https://github.com/psyb0t/ctxerrors). There is no config loader and no HTTP framework; every handler talks to `log/slog` and the standard library.

## Quick look

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
LOG_LEVEL=debug LOG_FORMAT=json go run main.go
```

```json
{"time":"2026-11-10T07:09:49.382091169Z","level":"INFO","msg":"this is an info message","user":"psyb0t","action":"testing"}
```

Info and debug records go to stdout, warnings and errors to stderr. Stdlib slog sends everything to stderr; the split here lets a container log collector keep error noise apart from happy-path noise without parsing anything.

## How the pieces fit

| Package | What it is |
| --- | --- |
| [`slogconf`](https://github.com/psyb0t/slogging/tree/main/slogconf) | Reads the environment and installs the default logger. Holds the calls that add sinks or swap the output. |
| [`handlers`](https://github.com/psyb0t/slogging/tree/main/handlers) | The structural handlers. `Handler` writes the process's own output, split by level. `FanOutHandler` tees one record to many handlers. |
| [`handlers/logring`](https://github.com/psyb0t/slogging/tree/main/handlers/logring) | A destination. A bounded ring of recent records with a search API. |
| [`handlers/loki`](https://github.com/psyb0t/slogging/tree/main/handlers/loki) | A destination. Pushes each record to Loki's HTTP push API. |

The split is the whole design. `slogconf` installs a `FanOutHandler` whose first slot is the output `Handler`. You change that one output with `SetOutput`, and you add as many destinations as you like with `AddSink`. A broken destination never silences the others. The fan-out hands each record to every handler that wants its level, keeps going past failures, and joins the errors.

## Where to go next

| You want to | Read |
| --- | --- |
| Set the level, format and source reporting, rename the variables, add a sink or replace the output | [Configuring slog](slogconf.md) |
| Write to other writers, move the stdout and stderr split, change the level at runtime, tee to several handlers | [Output and fan-out handlers](handlers.md) |
| Keep recent logs in memory and search them by attribute, level, time or text | [Searchable ring](logring.md) |
| Ship records to Grafana Loki | [Loki handler](loki.md) |
| Fix a startup panic, doubled lines, or records that never arrive | [Troubleshooting](troubleshooting.md) |
| Move from `slog-configurator` or the pre-1.7 handler API | [Migrating from slog-configurator](migrating.md) |
| Build, lint and test the library itself | [Development](development.md) |

The Go package docs are on [pkg.go.dev](https://pkg.go.dev/github.com/psyb0t/slogging), and the source is on [GitHub](https://github.com/psyb0t/slogging).
