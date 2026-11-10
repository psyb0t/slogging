# Documentation

Prefix: DOC.

- **DOC1:** A behavior, API, default, environment variable or build change updates the page that owns it in the same change: `docs/slogconf.md`, `docs/handlers.md`, `docs/logring.md`, `docs/loki.md`, `docs/troubleshooting.md`, `docs/migrating.md` or `docs/development.md`.
- **DOC2:** `docs/` stays flat; a file name is its URL.
- **DOC5:** Document what the code does today, including the failure behavior that looks bad (the Loki handler's synchronous push and silent drops).
