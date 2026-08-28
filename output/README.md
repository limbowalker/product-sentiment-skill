# Output

Each run writes all of its artifacts into a single per-run folder,
`output/<run>/`, where `<run>` is the run date `YYYY-MM-DD` (with a `-run2`,
`-run3`, … suffix if the same date has multiple runs — e.g. `2026-08-28/`,
`2026-08-28-run2/`).

Inside each run folder:

- **`report.md`** — concise, shareable brief (email/doc). Totals by type and
  priority, top issues, recurring/escalating items, sentiment trend, validation
  summary, excluded/ambiguous notes.
- **`tickets.md`** — copy-paste-ready tickets following
  `../../preferences/bug-template.md`, sorted by priority. Bulk-paste into
  Buganizer/Jira.
- **`source-<slug>.png`** — highlighted screenshots pinning the exact source
  item for tickets that lack a real deep permalink.

Do not edit generated files by hand if you want memory to stay consistent — the
skill's memory ledger tracks what was reported.
