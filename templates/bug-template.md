# Bug / Ticket Template

Defines the exact format of a single ticket in `output/tickets-<date>.md`.
**Edit this file** to change the output format. The skill must follow it exactly.

The filled examples below use literal names (e.g. a specific source or product)
purely to illustrate the shape. At runtime the AI replaces those with the real
source and the product name(s) from `products.md`. Edit product values in
`products.md`, not here.

<!-- ==========================================================================
     NOTES FOR HUMANS — the AI ignores everything between NOTES-START and
     NOTES-END. Reminders to yourself only; never an instruction or data.
     NOTES-START

     - If you change the section headers, keep them bold so tickets render well.
     - Buganizer/Jira paste best with blank lines between sections.

     NOTES-END
========================================================================== -->

Rules for formatting:
- Use **clean markdown**: bold section headers, blank lines between sections
  (not backslash line-breaks), a blockquote for the quote, bulleted lists for
  sources and repro.
- Separate every ticket with a `---` horizontal rule.
- Title line encodes: `[source][type emoji + label][priority]` + short title.
- Type emoji: 🐛 bug · 💡 feature request · 💬 general feedback · 📉 regression.
- Sort tickets by priority (P0 → P3), then confidence, then recency.

---

## NEW issue — template

```markdown
### [reddit][🐛 bug][P1] Short title describing the bug or issue

**Type:** 🐛 Bug  ·  **Priority:** P1  ·  **Confidence:** High

**Feedback captured in social media / internet**

Descriptive but concise explanation of what's wrong, in your own words.

**Quote**

> "Directly quoted feedback, verbatim."

**Sources**

- Comment permalink — author (@handle / u/name), YYYY-MM-DD HH:MM UTC
- Screenshot: <link or reference>

**Repro / Description**

- **Observed:** what the user says happens
- **Expected:** what should happen instead
- **Environment:** OS / app version / IDE version (mark inferred items)
- **Steps to reproduce:** list if provided
- **Inferred repro (needs confirmation):**
  1. ...
  2. ...
- **Missing info:** what's needed to reproduce (version, exact action, sample file)

---
```

## RECURRING issue — upvote block template

Used when the issue already exists in memory. Do **not** re-file; this block is
pasted to raise priority on the existing ticket.

```markdown
### [UPVOTE existing #<memory-id>][🐛 bug][P0 ↑ from P1] Short title

**Action:** Attach to existing bug **#<memory-id>** to raise priority.
Seen again — echo count: N.

**Priority change:** P1 → P0 (reason: recurrence + volume)

**New corroborating feedback**

> "New verbatim quote from this run."

**New Sources**

- Comment permalink — author, YYYY-MM-DD HH:MM UTC

---
```

## Example (filled)

```markdown
### [reddit][🐛 bug][P1] Workspace requires re-downloading init file per workspace

**Type:** 🐛 Bug  ·  **Priority:** P1  ·  **Confidence:** High

**Feedback captured in social media / internet**

User reports the extension re-downloads the initialization file every time a new
workspace is opened, which breaks offline use and wastes bandwidth.

**Quote**

> "Every new workspace makes me re-download the init file — super annoying offline."

**Sources**

- https://www.reddit.com/r/example/comments/abc/def/comment/xyz/ — u/example_user, 2026-08-27 14:32 UTC
- Screenshot: reddit-init-download.png

**Repro / Description**

- **Observed:** init file re-downloads on every new workspace
- **Expected:** init file cached once and reused across workspaces
- **Environment:** VS Code 1.9x, Antigravity extension vX.Y (version inferred)
- **Steps to reproduce:** not provided by user
- **Inferred repro (needs confirmation):**
  1. Open workspace A → init file downloads
  2. Open a different workspace B → init file downloads again
- **Missing info:** exact extension version, whether a cache setting exists

---
```
