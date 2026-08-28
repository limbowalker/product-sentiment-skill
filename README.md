# 📡 Product Sentiment Tracker
👤 **Author:** <a href="https://ux.benshishko.eu/" target="_blank" rel="noopener">Ben Shishko</a>


> Turn the internet's noise about your product into a tidy stack of tickets. 🎯

An agent skill that runs a customer-sentiment report for a product. It scans the
social media and review sources you configure for 🐛 bugs, 💡 feature requests,
and 💬 complaints, then hands you prioritized, deduplicated, reference-validated
tickets ready to bulk copy-paste into Buganizer or Jira.

## ✨ What you get

Every run produces two artifacts:

- **`report.md`** — a concise, shareable brief (totals by type/priority, top
  issues, recurring items, sentiment trend, validation summary).
- **`tickets.md`** — copy-paste-ready tickets, one per atomic issue, following
  *your own* bug template.

Plus the smart bits under the hood:

- 🧠 **Memory that escalates** — an append-only ledger so recurring, unfixed
  issues climb in priority instead of getting re-filed as duplicates.
- 🔗 **No hallucinated links** — every reference is fetched and verified before a
  single ticket is written.
- 📸 **Pin-point screenshots** — when a source has no permalink, it captures a
  highlighted screenshot of the exact review.


## 🚀 Install

This is an agent skill. Rather than list steps for every agent (opencode,
Antigravity, Claude, etc.), just copy the prompt below and hand it to your
favorite agent — it'll clone and install the skill for you. 🪄

```
Install the agent skill from https://github.com/limbowalker/product-sentiment-skill
into my agent's skills directory, following that repo's instructions.
```


## ⚙️ Configure

The skill reads three plain-text preference files (in `preferences/`) so you can
tweak behavior without touching the skill itself:

- 🏷️ `preferences/products.md` — the product name(s) to track.
- 🌐 `preferences/sources.md` — the URLs to scan (empty? it falls back to an
  automatic web search).
- 📝 `preferences/bug-template.md` — the output format for a single ticket.

> 🔒 **Your `preferences/` stay private.** The whole `preferences/` directory is
> git-ignored, so your product names and source URLs are never committed. The
> repo ships **templates with identical filenames** in `templates/`. On first
> run the skill copies `templates/*` → `preferences/*` (same paths, so nothing
> else changes) and then interviews you to fill them in. To reset a file, delete
> it from `preferences/` and re-run.

## ▶️ Run

Trigger it from your agent with a prompt like:

```
Run the product sentiment report using the product-sentiment-skill skill.
```

📂 Output lands in a per-run folder: `output/<YYYY-MM-DD>/`.

## ⏰ Scheduling (optional)

Want it running on autopilot? For macOS `launchd` or `cron`, see
`scheduling/README.md`. Just edit the paths in the provided config to point at
wherever you installed the skill (e.g. `~/path/to/product-sentiment-skill`).
