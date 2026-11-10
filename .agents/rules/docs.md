# Documentation

Prefix: DOC. The README is the front door; `docs/` is the documentation, published at https://psyb0t.github.io/slogging/ from `main`.

- **DOC1:** A behavior, API, default, environment variable or build change updates the page that owns it in the same change: `docs/slogconf.md`, `docs/handlers.md`, `docs/logring.md`, `docs/loki.md`, `docs/troubleshooting.md`, `docs/migrating.md` or `docs/development.md`. Reference material lives in `docs/`, not in the README.
- **DOC2:** Every page has front matter with a unique `title` and a `description` of 50 to 170 characters, and exactly one `#` heading. `docs/` stays flat; a file name is its URL.
- **DOC3:** Page-to-page links use the relative `.md` path. Links to files outside `docs/`, and every repository link in the README, use the full `https://github.com/psyb0t/slogging/blob/main/...` (or `tree/main/...`) URL. README links to documentation use the full site URL.
- **DOC4:** `make docs` builds the site with GitHub's Pages image. Build it after any change under `docs/` and fix broken links before committing.
- **DOC5:** Document what the code does today, including the failure behavior that looks bad (the Loki handler's synchronous push and silent drops). Copy error text from the source. Run the examples you add.
- **DOC6:** Prose: one paragraph per source line, no em or en dashes, "directory" not "folder", plain words instead of AI vocabulary, sentence-case headings, first person singular when a person speaks. Public text never names local scratch directories.
