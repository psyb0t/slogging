---
title: "Output and fan-out handlers"
description: "handlers.Handler splits records between stdout and stderr writer sets by level; FanOutHandler tees each record to several slog handlers."
nav_order: 3
---

# Output and fan-out handlers

[`handlers`](https://github.com/psyb0t/slogging/tree/main/handlers) holds the two structural pieces a chain is built from. `Handler` writes the process's own output. `FanOutHandler` passes one record to several handlers. The destinations, [the ring](logring.md) and [Loki](loki.md), live in subpackages.

## Handler

`Handler` sends every record to one of two writer sets, chosen by level. `NewStd` builds the usual one, stdout for informational records and stderr for warnings and above:

```go
import "github.com/psyb0t/slogging/handlers"

h, err := handlers.NewStd(handlers.Options{Format: handlers.FormatJSON})
if err != nil {
	panic(err)
}

slogconf.SetOutput(h)
```

`New` takes the writers as options. `Stdout` and `Stderr` are variadic and can be passed more than once:

```go
h, err := handlers.New(
	handlers.Options{Format: handlers.FormatJSON},
	handlers.Stdout(os.Stdout, logFile),
	handlers.Stderr(os.Stderr),
)
```

A side you leave out defaults to the matching `os` stream. Several writers on one side are joined with `io.MultiWriter`, so they all get the same bytes. If you want a different rendering per destination, such as text on the terminal and JSON in a file, build one `Handler` for each and tee them with a [`FanOutHandler`](#fanouthandler).

### Options

| Field | Meaning | Zero value means |
| --- | --- | --- |
| `Format` | `handlers.FormatJSON` or `handlers.FormatText` | `handlers.DefaultFormat`, which is text |
| `Level` | Minimum level emitted, a `slog.Leveler` | `slog.LevelInfo` |
| `AddSource` | Attach file, line and function to every record | off |
| `SplitAt` | Level at and above which records go to the stderr side, a `slog.Leveler` | `handlers.DefaultSplitLevel`, which is `slog.LevelWarn` |

Any other format string makes `New` and `NewStd` return an error wrapping `handlers.ErrInvalidFormat`.

**Point both sides at the same writer and everything lands together**, which is what stdlib slog does: it puts every level on stderr. Splitting is the default here so a log collector can separate error noise from happy-path noise without parsing.

```go
h, err := handlers.New(
	handlers.Options{},
	handlers.Stdout(os.Stderr),
	handlers.Stderr(os.Stderr),
)
```

### Changing the level at runtime

`Options.Level` is the one setting that stays live. It is a `slog.Leveler` that the handler reads on every record instead of once at construction, so a `*slog.LevelVar` you change later takes effect at once:

```go
level := new(slog.LevelVar)

h, err := handlers.NewStd(handlers.Options{Level: level})
if err != nil {
	panic(err)
}

slogconf.SetOutput(h)

level.Set(slog.LevelDebug) // takes effect immediately
```

`SplitAt` is read per record in the same way. Format, writers and `AddSource` are fixed once the handler is built.

## FanOutHandler

`FanOutHandler` holds a list of `slog.Handler` values and passes each record to all of them. It holds the interface, not this package's `Handler`, so the ring, the Loki handler and any handler you wrote yourself can share one chain.

```go
console, _ := handlers.NewStd(handlers.Options{})
file, _ := handlers.New(
	handlers.Options{Format: handlers.FormatJSON},
	handlers.Stdout(logFile),
	handlers.Stderr(logFile),
)

slog.SetDefault(slog.New(handlers.NewFanOut(console, file)))
```

How it behaves:

- `Enabled` is true when any child wants the level. A ring keeping `debug` beside a console at `info` is a normal setup, and the fan-out must not veto a record one child asked for.
- `Handle` passes the record to each child whose own `Enabled` is true for that level, so the noisier child does not force output on the quieter one.
- A failing child does not stop the rest. Every enabled child gets the record, and the failures come back joined into one error, each wrapped with `handler N failed`. This matters because slog throws away whatever `Handle` returns. A fan-out that stopped at the first error would let one broken destination silence everything after it, with nothing saying why.
- `WithAttrs` and `WithGroup` apply to every child.
- `Len` reports how many children it has. `Handlers` returns a copy of the list, so changing the copy does not change where the process logs.

`slogconf` builds its default logger on a fan-out whose first slot is the output `Handler`. Its [`AddSink` and `SetOutput`](slogconf.md#adding-a-destination-or-changing-the-output) calls work on that list.
