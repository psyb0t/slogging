---
title: "Searchable ring"
description: "logring keeps recent slog records in a byte-bounded in-memory ring and searches them by attribute, level, time range, substring or regexp."
nav_order: 4
---

# Searchable ring

[`handlers/logring`](https://github.com/psyb0t/slogging/tree/main/handlers/logring) keeps the most recent records in a bounded in-memory ring, so a process can answer "what just happened" without leaving the process. A debug endpoint or an admin page can read it directly.

The ring is a `slog.Handler`. On its own it captures records and prints nothing, so add it next to your output instead of replacing it:

```go
import (
	"github.com/psyb0t/slogging/handlers/logring"
	"github.com/psyb0t/slogging/slogconf"
)

ring := logring.New(logring.Options{})
slogconf.AddSink(ring)
```

**It is a debugging aid, not a log store.** It is per process, bounded, and gone when the process dies, which is exactly when you most want the logs. Ship them somewhere durable as well, for example with the [Loki handler](loki.md).

## Options

The zero value works. Every field falls back to a `Default*` constant.

| Field | Meaning | Zero value means |
| --- | --- | --- |
| `MaxBytes` | Total size of retained records. Oldest records are evicted to stay under it. | `DefaultMaxBytes`, 100 MiB |
| `MaxRecordBytes` | A single record larger than this is dropped and counted. Clamped to `MaxBytes`. | `DefaultMaxRecordBytes`, 1 MiB |
| `Level` | Minimum level retained, a `slog.Leveler` | `DefaultLevel`, `slog.LevelInfo` |
| `Text` | Store lines in slog's text format instead of JSON | JSON |

The ring is bounded **by bytes, not record count**, so one huge line cannot evict a hundred useful ones. A record's size is its formatted line plus its message plus the keys and values of its captured attributes, since all of that stays in memory.

The default level skips `debug`. A service that logs every SQL statement at `debug` would fill the ring with query traces in seconds. The ring applies its own level even inside a fan-out, so a process running at `debug` does not pour its trace lines into a ring set to `warn`.

`Text` changes only the format of `Entry.Line`. Attribute search reads the captured attributes, so it works the same in either format.

## What an entry holds

```go
type Entry struct {
	Time  time.Time
	Level slog.Level
	Msg   string
	Line  string // the formatted record, without the trailing newline
	Attrs []Attr // Key and Value, both strings
}
```

`Attrs` holds the record's attributes plus the ones bound earlier with `logger.With(...)`, which never appear on the `slog.Record` itself. Grouped attributes get dotted keys: a logger from `WithGroup("http")` logging `status` stores `http.status`. Values are rendered to strings when the record is captured, so a `slog.LogValuer` keeps what it returned at log time and the ring does not hold on to your objects. `entry.Attr("request_id")` returns one value and whether it was there.

`Line` does not include source information, even when the process output does.

## Searching

```go
page := ring.Search(logring.SearchOptions{
	Attrs:    map[string]string{"request_id": "abc123"},
	MinLevel: slog.LevelWarn,
	Limit:    50,
})

fmt.Printf("showing %d of %d\n", len(page.Entries), page.Total)
```

`Search` returns a `Page`:

| Field | Meaning |
| --- | --- |
| `Entries` | The matches for this page, newest first unless `Ascending` is set |
| `Total` | How many entries matched before `Limit` and `Offset` were applied |
| `Offset` | The offset you asked for, echoed back |

Without `Total`, a full page and the last page look the same. It is counted in the same locked walk that collects the page, so both numbers describe the same moment of the ring. The price is that every search visits every entry, even after the page is full.

The zero `SearchOptions{}` returns everything, newest first, capped at `DefaultSearchLimit` (200). Every filter you set has to pass:

| Field | Keeps |
| --- | --- |
| `Contains` | Entries whose `Line` contains this substring, ignoring case |
| `Exclude` | Drops entries whose `Line` contains this substring, ignoring case |
| `Match` | Entries whose `Line` matches this compiled `*regexp.Regexp` |
| `Attrs` | Entries carrying every one of these key and value pairs exactly, matched against the captured attributes, not the line |
| `MinLevel` | Entries at or above this `slog.Leveler`. Nil applies no level filter, so custom levels below `debug` still come back. |
| `Levels` | Entries at exactly one of these levels, for "warnings and errors but not info". Empty applies no filter. |
| `Since` | Entries at or after this time |
| `Until` | Entries at or before this time |
| `Limit` | At most this many entries. Zero or less means 200. |
| `Offset` | Skips this many matches first, for paging |
| `Ascending` | Oldest first instead of newest first |

`Match` takes a compiled expression, so a bad pattern fails where you compile it with `regexp.MustCompile` or `regexp.Compile`, not inside `Search`.

To page through a large result, keep `Limit` fixed and raise `Offset` by `Limit` until `Offset` reaches `Total`. Records arriving between calls shift the pages.

## Other reads

```go
matches := ring.Count(opts)   // matches only; Limit and Offset are ignored
recent := ring.Tail(50)       // newest 50, oldest first, unfiltered
bytes := ring.Size()          // bytes held: the number the ring bounds itself by
count := ring.Len()           // records held
n, b, dropped := ring.Stats() // records, bytes and drops under one lock
ring.Clear()                  // empty the ring
```

Watch `Size()` against `MaxBytes` to see how close the ring is to evicting. A nonzero `dropped` means some records were refused for exceeding `MaxRecordBytes`, so a search is running over an incomplete picture. `dropped` counts for the life of the ring and survives `Clear`.

Handlers derived with `With` or `WithGroup` write into the same ring, and every method is safe for concurrent use.
