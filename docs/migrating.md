---
title: "Migrating from slog-configurator"
description: "Move from github.com/psyb0t/slog-configurator to slogging, and from the pre-1.7 AddHandler and MultiWriterHandler API to AddSink and handlers.Handler."
nav_order: 7
---

# Migrating from slog-configurator

This module was `github.com/psyb0t/slog-configurator` through v1.5.0. Those versions keep resolving under the old path, so nothing breaks until you move. The [CHANGELOG](https://github.com/psyb0t/slogging/blob/main/CHANGELOG.md) has the full story of each release.

## Import paths

| Before | After |
| --- | --- |
| `_ "github.com/psyb0t/slog-configurator"` | `_ "github.com/psyb0t/slogging/slogconf"` |
| `slogconfigurator.Init(...)` | `slogconf.Init(...)` |
| `github.com/psyb0t/slog-configurator/logring` | `github.com/psyb0t/slogging/handlers/logring` |
| `github.com/psyb0t/common-go/slogging/loki` | `github.com/psyb0t/slogging/handlers/loki` |

Every exported name was unchanged in that move. Only import paths and the package name changed, so it was a find-and-replace on imports.

## The handler API since v1.7.0

| Before | After |
| --- | --- |
| `slogconf.AddHandler(sink)` | `slogconf.AddSink(sink)` |
| none | `slogconf.SetOutput(h)`, which swaps where you print and keeps the sinks |
| `slogconf.NewMultiWriterHandler(fmt, opts, out, err)` | `handlers.New(handlers.Options{...}, handlers.Stdout(out), handlers.Stderr(err))` |
| `slogconf.MultiWriterHandler` | `handlers.Handler` |
| `slogconf.FanOutHandler` / `NewFanOutHandler` | `handlers.FanOutHandler` / `handlers.NewFanOut` |

`MultiWriterHandler` never was one. It took exactly two writers and routed between them by level. `handlers.Handler` is that thing named for what it does, and `handlers.Stdout` and `handlers.Stderr` are variadic, so several writers per stream is now something the API offers. See [Output and fan-out handlers](handlers.md).

The Loki handler also dropped its `gonfiguration` and `common-go` dependencies. `NewClient` and `NewHandler` read the same `SLOGGING_LOKI_URL` and `SLOGGING_LOKI_APPNAME` variables as before, through the standard library.
