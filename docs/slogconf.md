---
title: "Configuring slog"
description: "slogconf reference: LOG_LEVEL, LOG_FORMAT and LOG_ADD_SOURCE, renaming them with Options, and AddSink, SetOutput and SetHandlers for the handler chain."
nav_order: 2
---

# Configuring slog

[`slogconf`](https://github.com/psyb0t/slogging/tree/main/slogconf) configures slog's default logger from environment variables. Importing it is enough: its `init` reads the environment and installs the logger before your `main` runs.

```go
import _ "github.com/psyb0t/slogging/slogconf"
```

## Environment variables

| Variable | Values | Default |
| --- | --- | --- |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error`, in any case | `info` |
| `LOG_FORMAT` | `json`, `text`, in any case | `text` |
| `LOG_ADD_SOURCE` | anything Go's `strconv.ParseBool` accepts: `1`, `t`, `true`, `TRUE`, `0`, `f`, `false` and so on | `false` |

A variable that is set but empty counts as unset, so `LOG_LEVEL=` left in a shell profile falls back to the default instead of failing. The names are exported as `slogconf.EnvVarNameLevel`, `EnvVarNameFormat` and `EnvVarNameAddSource`.

An invalid value is a startup failure. The blank import panics with the error, and `Init` returns it. The level is checked first, so with a bad level and a bad format at once you see the level error. The three sentinels are `ErrInvalidLogLevel`, `ErrInvalidLogFormat` and `ErrInvalidLogAddSource`, and `errors.Is` matches them through the wrapping. The exact messages are in [Troubleshooting](troubleshooting.md#the-process-panics-at-startup).

With the level at `debug`, configuration ends with one line saying what it picked:

```json
{"time":"2026-11-10T07:09:49.381960536Z","level":"DEBUG","msg":"slogconf: configured","level":"debug","format":"json","addSource":false}
```

## Output streams

The installed output sends `debug` and `info` records to stdout and `warn` and `error` records to stderr. To change the writers or the split point, build your own [`handlers.Handler`](handlers.md) and install it with `SetOutput`.

## Your own variable names

`LOG_LEVEL` and friends are the defaults. Call `Init` with `Options` to read other variables:

```go
import "github.com/psyb0t/slogging/slogconf"

if err := slogconf.Init(slogconf.Options{
	LevelEnvVar:  "MYAPP_LOG_LEVEL",
	FormatEnvVar: "MYAPP_LOG_FORMAT",
}); err != nil {
	panic(err)
}
```

Only the variables you name get read, so a stray `LOG_LEVEL` cannot override `MYAPP_LOG_LEVEL`. A field you leave empty keeps its default name.

| Field | Meaning | Empty means |
| --- | --- | --- |
| `LevelEnvVar` | Variable read for the level | `LOG_LEVEL` |
| `FormatEnvVar` | Variable read for the format | `LOG_FORMAT` |
| `AddSourceEnvVar` | Variable read for source reporting | `LOG_ADD_SOURCE` |
| `DefaultLevel` | Level used when the variable is unset or empty. Validated like the variable. | `info` |
| `DefaultFormat` | Format used when the variable is unset or empty. Validated like the variable. | `text` |
| `DefaultAddSource` | Source reporting when the variable is unset or empty | `false` |

`slogconf.Init(slogconf.Options{})` does exactly what the blank import does. The blank import still runs first whenever the package is imported, so your environment has to hold valid values for the default names too, or the process panics before `Init` is reached.

**Call `Init` early.** It replaces slog's default logger and the whole handler chain, so it also discards any sink added before it. A logger derived with `slog.Default().With(...)` before `Init` keeps writing through the old chain.

## Adding a destination or changing the output

These are different jobs with different calls:

```go
slogconf.AddSink(ring)      // ALSO send records here: appends
slogconf.SetOutput(handler) // send my output THERE instead: replaces, keeps sinks
slogconf.SetHandlers(h)     // start over: replaces the output AND every sink
```

`slogconf` installs a [`handlers.FanOutHandler`](handlers.md#fanouthandler) whose first slot is the output handler. `AddSink` appends to it. `SetOutput` replaces the first slot and leaves the sinks where they were.

The difference matters. The output already writes to stdout and stderr. Add a second terminal handler with `AddSink` and both receive every record, so every line prints twice. `SetOutput` is the call that swaps it.

`SetHandlers(hs...)` replaces everything with a new fan-out over `hs`. It is the escape hatch, mostly for tests that point everything at a buffer or `io.Discard`.

`AddSink` and `SetOutput` return `false` when slog's default is no longer a fan-out this package installed, for example because other code called `slog.SetDefault`. The call still takes effect. `AddSink` wraps the foreign handler and the sink in a new fan-out. `SetOutput` replaces the whole chain with your handler. Either way the stdout and stderr split from `slogconf` is gone, which is worth checking for rather than discovering through missing logs:

```go
if !slogconf.AddSink(ring) {
	slog.Warn("default logger was replaced elsewhere; the ring is stacked on it")
}
```

Like `Init`, all three replace slog's default logger, and loggers derived earlier with `With` or `WithGroup` keep the chain they captured.

## Changing the level at runtime

The output that `slogconf` builds uses a fixed level. For a level you can change while the process runs, build a [`handlers.Handler`](handlers.md#changing-the-level-at-runtime) with a `*slog.LevelVar` and install it with `SetOutput`.
