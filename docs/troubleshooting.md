---
title: "Troubleshooting"
description: "Fixes for slogging problems: the import-time panic on a bad LOG_LEVEL, doubled log lines, sinks that see nothing, Loki records that never arrive."
nav_order: 6
---

# Troubleshooting

## The process panics at startup

The [`slogconf`](slogconf.md) blank import validates the environment in `init` and panics on a bad value, before `main` runs. The message names the variable's value and the file and line that rejected it:

```
panic: failed to get log level: verbose: invalid log level [.../slogconf/level.go:31 in github.com/psyb0t/slogging/slogconf.getSlogLevel] [...]
```

```
panic: yaml: invalid log format [.../slogconf/format.go:23 in github.com/psyb0t/slogging/slogconf.parseFormat]
```

```
panic: LOG_ADD_SOURCE="maybe": invalid log add-source value [.../slogconf/log.go:203 in github.com/psyb0t/slogging/slogconf.readConfig]
```

Fix the variable. `LOG_LEVEL` takes `debug`, `info`, `warn` or `error`. `LOG_FORMAT` takes `json` or `text`. `LOG_ADD_SOURCE` takes what `strconv.ParseBool` accepts, such as `true` or `0`. An empty value is fine; it means the default.

This also happens when you use your own names with `slogconf.Init`: the blank import still reads the default names first, so a bad `LOG_LEVEL` panics even if your code reads `MYAPP_LOG_LEVEL`.

## Every line prints twice

You added a second handler that writes to the terminal with `AddSink`. The output `slogconf` installed still writes to stdout and stderr, and both handlers get every record. Use `slogconf.SetOutput(h)` to replace the output instead. See [Adding a destination or changing the output](slogconf.md#adding-a-destination-or-changing-the-output).

## A sink or a new output sees nothing from some loggers

`slog.Logger.With` and `WithGroup` capture the handler chain at the moment you call them. A logger derived before `Init`, `AddSink`, `SetOutput` or `SetHandlers` keeps writing through the old chain. Configure logging first in `main`, then derive loggers.

`Init` also replaces the whole chain, so a sink added before `Init` is gone afterwards. Call `Init` first, then `AddSink`.

## AddSink or SetOutput returns false

Something else replaced slog's default logger with `slog.SetDefault`, so it is no longer the fan-out `slogconf` installed. The call still worked, but the stdout and stderr split is gone. Find the other `slog.SetDefault` call, or call `slogconf.Init` after it to put the `slogconf` chain back.

## Loki receives nothing

Records that fail to push are dropped. To see why, run with `LOG_LEVEL=debug` and look for `loki push failed` in the output. A Loki that answers with an error status, such as a 400 for a rejected payload, produces no line at all. Check Loki's own logs for those.

Also check that `SLOGGING_LOKI_URL` is the base URL, such as `http://loki:3100`, without `/loki/api/v1/push`. The client appends the path itself. See [Loki handler](loki.md).

## The process dies with a stack overflow after adding the Loki handler

The Loki handler is set to `slog.LevelDebug` and the push is failing. Each failure logs a `debug` line through the default logger, which the Loki handler tries to push, which fails again. Set the handler to `slog.LevelInfo` or higher. See [Failure behavior](loki.md#failure-behavior).

## Logging calls are slow

The Loki handler makes one HTTP request per record inside the logging call, with a 5 second timeout. A slow or unreachable Loki slows down every call that reaches it. Raise the handler's level to cut the number of records it sends, or fix the path to Loki.

## The ring is missing records

Check `ring.Stats()`. A nonzero third value means records were dropped for exceeding `MaxRecordBytes`. If `Size()` sits near `MaxBytes`, older records are being evicted to make room. Raise `MaxBytes`, or raise the ring's `Level` so it keeps fewer records. The ring skips `debug` by default. See [Searchable ring](logring.md#options).
