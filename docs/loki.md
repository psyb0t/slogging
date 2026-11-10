---
title: "Loki handler"
description: "The loki package pushes slog records to Grafana Loki's HTTP push API: SLOGGING_LOKI_URL and SLOGGING_LOKI_APPNAME, labels, line format and failure behavior."
nav_order: 5
---

# Loki handler

[`handlers/loki`](https://github.com/psyb0t/slogging/tree/main/handlers/loki) is a `slog.Handler` that pushes each record to [Grafana Loki](https://grafana.com/oss/loki/) through its HTTP push API, `POST /loki/api/v1/push`. Add it as a sink next to your normal output:

```go
import (
	"github.com/psyb0t/slogging/handlers/loki"
	"github.com/psyb0t/slogging/slogconf"
)

client, err := loki.NewClient() // reads SLOGGING_LOKI_URL
if err != nil {
	panic(err)
}

handler, err := loki.NewHandler( // reads SLOGGING_LOKI_APPNAME
	client,
	slog.LevelInfo,
	map[string]bool{"tenant": true}, // these attrs become Loki labels
)
if err != nil {
	panic(err)
}

slogconf.AddSink(handler)
```

## Settings

| Variable | Read by | Meaning |
| --- | --- | --- |
| `SLOGGING_LOKI_URL` | `NewClient` | Loki base URL without the push path, such as `http://loki:3100`. A trailing slash is trimmed. Required. |
| `SLOGGING_LOKI_APPNAME` | `NewHandler` | Value of the `app` label on every stream. Required. |

An empty value makes the constructor return an error that contains `loki URL is required` or `loki app name is required`. The names are exported as `loki.EnvVarNameURL` and `loki.EnvVarNameAppName`.

To skip the environment, pass the settings directly:

```go
client, err := loki.NewClientWithConfig(loki.ClientConfig{
	URL: "http://loki:3100",
})

handler, err := loki.NewHandlerWithConfig(client, loki.HandlerConfig{
	AppName:   "billing",
	LabelKeys: map[string]bool{"tenant": true},
}, slog.LevelInfo)
```

One `Client` can serve several handlers. The handler's level is a fixed `slog.Level`; it does not take a `slog.LevelVar`.

## What Loki receives

Each record becomes one stream entry with a nanosecond timestamp.

Labels on every stream:

| Label | Value |
| --- | --- |
| `app` | The configured app name |
| `level` | The record level in lowercase: `debug`, `info`, `warn`, `error` |
| `service` | The record's `service` attribute when `service` is in `LabelKeys`, otherwise `system` |
| `source_file`, `source_func` | The file and function that logged the record, when slog captured a caller. Records logged through `slog.Logger` methods always have one. |

Any attribute whose key is in `LabelKeys` also becomes a label. Every other attribute goes into the log line as ` key=value` after the message, attributes bound with `With` first, then the record's own:

```
payment failed order=1234 amount=9.99
```

**Choose `LabelKeys` carefully.** Loki indexes by label, and every distinct label value creates a new stream, so a high-cardinality key like `request_id` makes one stream per request. The `source_file` and `source_func` labels already make one stream per logging call site.

Known limits of the current attribute handling: a group value such as `slog.Group("http", "status", 200)` lands in the line as `http=[status=200]` and is never matched against `LabelKeys`; with nested `WithGroup` calls the key prefix comes out in reverse order (`b.a.key` for `WithGroup("a").WithGroup("b")`); and attributes bound with `With` before a `WithGroup` call also get that group's prefix.

## Failure behavior

Each record is one synchronous HTTP request made inside `Handle`, with a 5 second client timeout and no retry. A slow Loki therefore slows down every logging call that reaches this handler, by up to 5 seconds each, and an unreachable one costs whatever the connection attempt takes.

Failures are dropped. A failed request is reported with a `debug` line through slog's default logger (`loki push failed`, `loki request build failed`, `loki payload marshal failed`). A response with an error status, such as a 400 or a 500, is not reported at all. `Handle` always returns nil, since slog discards whatever it returns anyway.

**Keep the handler's level above `debug`.** The failure lines go through slog's default logger, which includes this handler when it was added with `AddSink`. At `slog.LevelDebug`, a push that fails logs a `debug` line, which this handler tries to push, which fails again, until the process dies with a stack overflow.

## Lower-level client calls

`Client` can also push without slog:

```go
client.Push(ctx, map[string]string{"app": "billing"}, "a raw line")

client.PushLog(ctx, "billing", "worker", "INFO", "job done", map[string]any{
	"job": 42,
})
```

`PushLog` sets the `app`, `source` and lowercased `level` labels and appends each `data` entry to the line as ` key=value`, in Go's random map order. Both calls follow the failure behavior above.
