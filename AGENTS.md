# slogging

Instructions for coding agents (Codex, Claude Code and others) changing this repository. People using the library want the [README](README.md) and the documentation site at https://psyb0t.github.io/slogging/.

## Agent setup

`CLAUDE.md` is a committed symlink to `AGENTS.md`, and `.claude` is a committed symlink to `.agents`, so Claude Code reads the same instructions, rules and skills as Codex. Claude's Edit and Write tools refuse to write through a symlink, so edit `AGENTS.md` and `.agents/rules/*.md` directly.

## What it is

A Go library for the stdlib `log/slog`, module `github.com/psyb0t/slogging`. Importing `slogconf` configures slog's default logger from `LOG_LEVEL`, `LOG_FORMAT` and `LOG_ADD_SOURCE` at `init` time. The `handlers` packages add destinations to that logger: a searchable in-memory ring and a Loki pusher. It replaced `github.com/psyb0t/slog-configurator` (last release v1.5.0 under that path). There is no binary, no Docker image and no service.

## Architecture

- `slogconf`: `init` calls `configureWith(Options{})`, which reads the environment (`readConfig` in `log.go`), builds a `handlers.Handler` with `handlers.NewStd`, and installs it through `SetHandlers` as slot 0 of a `handlers.FanOutHandler` on `slog.Default()`. An invalid value panics in `init`; `Init(Options)` returns the same error instead. `handlers.go` holds the three chain calls: `SetOutput` replaces slot 0 (`outputSlot`), `AddSink` appends, `SetHandlers` replaces everything. The first two return false when the default is not a `*handlers.FanOutHandler`.
- `handlers`: `Handler` (`handler.go`) routes each record to a stdout or stderr writer set by `Options.SplitAt` (default `slog.LevelWarn`), with `Options.Level` resolved on every record. `FanOutHandler` (`fanout.go`) dispatches to every enabled child, continues past failures and joins the errors. `Format` (`format.go`) is `json` or `text`.
- `handlers/logring`: `Handler` writes into a shared `store` bounded by bytes (`DefaultMaxBytes` 100 MiB, `DefaultMaxRecordBytes` 1 MiB). It formats each record with an inner stdlib handler into a shared buffer under `fmtMu`, and captures attributes, including ones bound with `With`, as dotted-key strings. `Search` walks the whole store under one read lock to fill a `Page` and its `Total`.
- `handlers/loki`: `Client` posts to `<URL>/loki/api/v1/push` with a 5 second timeout. `Handler.Handle` builds labels (`app`, `level`, `service`, `source_file`, `source_func`, plus `LabelKeys`) and a `msg key=value` line and calls `Client.Push` synchronously. Failures are logged at debug through `slog.Default()` and dropped; `Handle` always returns nil.

Data flow: `slog.Info(...)` -> `slog.Default()` -> `FanOutHandler` -> slot 0 `handlers.Handler` (stdout or stderr) and each sink added with `AddSink` (`logring.Handler`, `loki.Handler`, or any `slog.Handler`).

## Stack

- Go, `go 1.26` in `go.mod`. Runtime dependency: `github.com/psyb0t/ctxerrors` only. Tests use `github.com/stretchr/testify`.
- golangci-lint v2 as a `tool` dependency in `go.mod`, run with `go tool golangci-lint`, configured in `.golangci.yml`.
- Dependencies are vendored in `vendor/` (`make dep` runs `go mod tidy` and `go mod vendor`).
- CI: `.github/workflows/pipeline.yml` calls the shared `go-workflow.yml`, `wait-for-default-branch.yml`, `release-workflow.yml` and `create-badges.yml` from `psyb0t/reusable-github-workflows`.
- Docs: Jekyll with the just-the-docs theme in `docs/`, published by GitHub Pages from `main`.

## Layout

| Path | Holds |
| --- | --- |
| `slogconf/` | `log.go` (env vars, `Options`, `Init`, `init`), `handlers.go` (`SetOutput`, `AddSink`, `SetHandlers`), `level.go`, `format.go`, `errors.go` |
| `handlers/` | `handler.go` (`Handler`, `Options`, `New`, `NewStd`, `Stdout`, `Stderr`), `fanout.go`, `format.go`, `errors.go` |
| `handlers/logring/` | `logring.go`: the ring, `Entry`, `SearchOptions`, `Page` |
| `handlers/loki/` | `loki.go`: `Client`, `Handler`, configs and env var names |
| `docs/` | The documentation site. `_config.yml`, one page per topic, `llms.txt`, `404.md`, the theme in `_sass/` and the tracking include in `_includes/` |
| `.github/workflows/` | `pipeline.yml` (CI and releases), plus `mirror-and-archive.yml`, `issue-pull.yml` and `collaborators-only.yml` |
| `.agents/rules/` | The rules below |
| `vendor/` | Vendored dependencies. Never edit by hand. |
| `CHANGELOG.md` | Release notes, newest first |

## Commands

Go 1.26 and Make, or only Docker:

```bash
make help           # every target
make lint           # go fix -diff, then golangci-lint
make lint-fix       # go fix, then golangci-lint --fix
make test           # go test -race ./...
make test-coverage  # tests with coverage, writes coverage-percent.txt, fails under 90%
make dep            # go mod tidy && go mod vendor
make docs           # build docs/ with the GitHub Pages image (needs Docker)
```

Without a local Go toolchain:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -v "$PWD:/src" \
  -w /src \
  golang:1.26 \
  make lint test-coverage
```

`pre-commit.sh` runs `make lint && make test-coverage`, the same pair CI runs.

## Rules

Read `.agents/rules/` before editing. One file per topic: Go code, handler behavior, the public API, tests, and docs.
