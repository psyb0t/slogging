# Handler behavior

Prefix: HND. These are the invariants the handlers are built around. Each one is pinned by tests; a change that breaks one is a breaking change (API rule API2).

- **HND1:** One failing handler never silences another. `FanOutHandler.Handle` calls every enabled child, keeps going after an error and returns `ctxerrors.Join` of the failures. slog discards what `Handle` returns, so stopping at the first error would drop output with nothing to show for it.
- **HND2:** `FanOutHandler.Enabled` is true when any child is enabled, and `Handle` re-checks each child's `Enabled`. A ring at `debug` beside a console at `info` must work both ways.
- **HND3:** A `slog.Leveler` passed in options is resolved on every call (`handlers.Handler.Enabled`, `logring.Handler.Enabled`, `SplitAt` in `Handle`), never stored as a resolved `slog.Level`, so a `*slog.LevelVar` keeps working.
- **HND4:** `WithAttrs` and `WithGroup` never write into a slice another handler can see. Copy, or append to `slices.Clip(...)`, as `logring` and `loki` do. Derived `logring` handlers share the parent's `store`, `fmtMu` and `fmtBuf` on purpose; see the `derive` comment.
- **HND5:** `slogconf` keeps the process output in slot 0 (`outputSlot`) of the fan-out. `SetOutput` replaces only that slot, `AddSink` only appends, `SetHandlers` replaces everything. Do not merge these calls or add one that both adds and replaces.
- **HND6:** `logring` is bounded by bytes, counts everything an `Entry` retains in `Entry.size`, clamps `MaxRecordBytes` to `MaxBytes`, and counts refused records in `dropped`. `Search` computes `Page.Total` in the same locked walk as the page. Stored values are strings, never `slog.Value`, so the ring pins no caller objects.
- **HND7:** A handler must not log through `slog.Default()` from inside `Handle` without a guard. The loki client does this for its debug failure lines today, and with the handler at `slog.LevelDebug` that recursion overflows the stack. New code must not add another path like it.
- **HND8:** `slogconf` validates the environment in `readConfig`, level before format before add-source, and an empty variable counts as unset (`envOrDefault`). Tests pin that order.
