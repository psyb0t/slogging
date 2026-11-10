---
title: "Development"
description: "Build, lint and test slogging with its Make targets, the 90 percent coverage gate, the vendored golangci-lint tool, the docs build, and how CI and releases run."
nav_order: 8
---

# Development

Hacking on slogging itself. You need Go 1.26 or newer and Make. Everything else, golangci-lint included, comes from the module's own `tool` dependencies in [`go.mod`](https://github.com/psyb0t/slogging/blob/main/go.mod) and is vendored in [`vendor/`](https://github.com/psyb0t/slogging/tree/main/vendor), so there is nothing to install globally. Without a local Go, run the same targets in the official image:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -v "$PWD:/src" \
  -w /src \
  golang:1.26 \
  make lint test-coverage
```

## Make targets

Run `make help` for the list. The [`Makefile`](https://github.com/psyb0t/slogging/blob/main/Makefile) has:

| Target | What it does |
| --- | --- |
| `make all` | Runs `dep`, `lint` and `test`, in that order. |
| `make dep` | `go mod tidy` and `go mod vendor`. |
| `make lint` | Fails if `go fix -diff ./...` has suggestions, then runs `go tool golangci-lint run ./...` with the rules in [`.golangci.yml`](https://github.com/psyb0t/slogging/blob/main/.golangci.yml). |
| `make lint-fix` | Applies `go fix ./...` and runs golangci-lint with `--fix`. |
| `make test` | `go test -race ./...`. |
| `make test-coverage` | Runs the tests with the race detector and a coverage profile, writes the total to `coverage-percent.txt`, and fails below 90 percent. |
| `make docs` | Builds this site locally with GitHub's Pages build image, into a gitignored directory. Needs Docker. |

[`pre-commit.sh`](https://github.com/psyb0t/slogging/blob/main/pre-commit.sh) runs `make lint && make test-coverage`, the same pair CI runs.

## Layout

| Path | Holds |
| --- | --- |
| [`slogconf/`](https://github.com/psyb0t/slogging/tree/main/slogconf) | Environment configuration, `Init`, and `AddSink`, `SetOutput`, `SetHandlers`. |
| [`handlers/`](https://github.com/psyb0t/slogging/tree/main/handlers) | `Handler` and `FanOutHandler`. |
| [`handlers/logring/`](https://github.com/psyb0t/slogging/tree/main/handlers/logring) | The searchable ring. |
| [`handlers/loki/`](https://github.com/psyb0t/slogging/tree/main/handlers/loki) | The Loki client and handler. |
| [`docs/`](https://github.com/psyb0t/slogging/tree/main/docs) | This site, built by GitHub Pages from the `main` branch. |

Most tests are internal (`package <name>`) and live next to the code in `*_internal_test.go`. [`slogconf/compat_test.go`](https://github.com/psyb0t/slogging/blob/main/slogconf/compat_test.go) is an external test that pins the call shapes downstream code compiles against, and [`handlers/logring/integration_test.go`](https://github.com/psyb0t/slogging/blob/main/handlers/logring/integration_test.go) checks the ring inside the configured fan-out.

## Docs site

Every page in `docs/` needs front matter with a unique `title` and a `description` of 50 to 170 characters. Link page to page with the relative `.md` path, and link files outside `docs/` with their full GitHub URL. `make docs` builds the site the way GitHub Pages does.

## CI

[`.github/workflows/pipeline.yml`](https://github.com/psyb0t/slogging/blob/main/.github/workflows/pipeline.yml) runs on every push to any branch or tag, once a week on a schedule, and by hand.

- On every branch push, the schedule and manual runs, the shared Go workflow runs `make lint`, `make test-coverage` and govulncheck.
- When that run is on `main`, the badges job then publishes the coverage, version, license and imported-by badges to the `badges` branch.
- On a `v*` tag, the pipeline does not rerun the checks. It waits for the `main` run of the same commit to pass, cuts the GitHub release, and refreshes the badges with the coverage from that `main` run. The `main` run keeps its coverage artifact for one day, so a tag pushed later than that still gets its release, but the badges job fails.

The weekly run exists for the imported-by badge, which pkg.go.dev updates on its own schedule.

## Releases

Versions follow [semver](https://semver.org), and the [CHANGELOG](https://github.com/psyb0t/slogging/blob/main/CHANGELOG.md) records every release. The Git tag is the version; there is no version file. A release is a commit on `main` that has passed CI, then an annotated `vX.Y.Z` tag on it.
