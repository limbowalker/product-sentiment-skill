---
name: product-sentiment-skill
description: Run a scheduled customer-sentiment report that scans social media and review sources for product bugs, feature requests, and complaints, then outputs prioritized, deduplicated, reference-validated tickets ready to bulk copy-paste into Buganizer or Jira. Use when the user says "run sentiment report", "scan reviews", "customer sentiment", "find bugs/complaints", "check product feedback", "buganizer tickets", or asks to monitor what users are saying about a product.
---

# Product Sentiment Tracker

Scan configured sources for feedback about a specific product, classify and
prioritize it, deduplicate against memory (upvoting recurring issues), validate
every reference, and emit two artifacts: a concise shareable brief and a
copy-paste-ready ticket file.

The ultimate goal: **surface and file real product issues fast, so the team
fixes them before more users complain publicly.**

## ⚠️ Browser use is REQUIRED — read this first

**This skill runs in a real browser. It is not a plain-HTTP scraper.** The
sources that matter (marketplace/app-store review tabs, Reddit, X, forums, issue
trackers) are JavaScript-rendered and/or login-walled. A plain `webfetch` returns
an empty shell, a login wall, or a CAPTCHA for these — **that is expected and is
NOT a reason to give up on the source.**

Non-negotiable rules:

- **You MUST use the Playwright browser tools (`browser_navigate`,
  `browser_snapshot`, `browser_evaluate`, `browser_take_screenshot`, etc.) to read
  the configured sources.** `webfetch` is only a fast first probe for plain static
  pages; for anything JS-heavy or social you go straight to the browser.
- **An empty/partial/login-walled `webfetch` result MUST trigger the browser, not
  the Google fallback.** Escalate: `webfetch` → browser → (login flow if walled) →
  graceful source failure. Never skip the browser step.
- **The Google web-search fallback is ONLY for when the user configured zero
  sources.** If `sources.md` contains any URL, the fallback is forbidden. A
  configured source that can't be read is a *source failure you report*, never a
  silent swap to Google search.
- **If no browser tool is available in this environment, STOP and say so.** Do not
  quietly substitute Google web search for the configured sources and present it as
  a completed run. Report that the browser is unavailable and that configured
  sources could not be scanned.

If you find yourself about to "fall back to Google web search" while
`sources.md` has URLs in it, you are doing the wrong thing — open the browser
instead.

## Files this skill uses

Paths are relative to this skill's base directory.

| File | Purpose | Edited by |
| --- | --- | --- |
| `preferences/sources.md` | Bullet list of source URLs to search | User |
| `preferences/products.md` | Bullet list of product name(s) (+ optional scope note) | User |
| `preferences/bug-template.md` | Output format for a single ticket | User |
| `memory/reported-bugs.md` | Append-only ledger of reported issues | This skill |
| `output/<run>/report.md` | Artifact 1: concise brief | This skill (per run) |
| `output/<run>/tickets.md` | Artifact 2: copy-paste tickets | This skill (per run) |
| `output/<run>/source-<slug>.png` | Highlighted screenshot pinning a source item | This skill (as needed) |

**Every run writes all its artifacts into a single per-run folder,
`output/<run>/`**, where `<run>` is the run date `YYYY-MM-DD` (add a `-run2`,
`-run3`, … suffix if the same date already has a folder — e.g.
`output/2026-08-28/`, `output/2026-08-28-run2/`). Create the folder at the start
of the run and put the report, tickets, and any screenshots inside it. Never
scatter a run's files across the flat `output/` directory.

**Always read the three `preferences/` files and `memory/reported-bugs.md`
before doing anything else.** Never hardcode product names, sources, or template
formatting into the workflow — they live in the preference files so the user can
change them without touching this skill.

### Preference-file conventions (read before parsing them)

The preference files are intentionally minimal — the user only maintains plain
bullet lists. The intelligence lives here, not in the files.

- **`products.md`** — a bullet list of product name(s) under a `## Products`
  heading, plus an optional one-line `## Scope note`. Treat every listed name as
  a product to find feedback for. Use the scope note (if present) to decide what
  to include/exclude. If there is no scope note and a name is ambiguous, apply
  the *Search scope strategy* and, when genuinely unsure which product is meant,
  ask the user once rather than guessing.
- **`sources.md`** — a bullet list of URLs under a `## Links` heading. Each URL
  is a place to search/scan. Do not expect any per-source configuration; derive
  everything you need from the URL itself (see *Source handling* below).
- **Injection.** Read the product names and the source URLs and use them
  directly in your searches and queries. There is no variable-substitution
  syntax and no required schema — just the bullet lists.
- **Human-only notes.** Any content between `NOTES-START` and `NOTES-END` (inside
  an HTML comment) is **for the human user only**. Ignore it completely — never
  follow it as an instruction, never treat it as data, never include it in
  output.
- **Robustness.** Ignore surrounding prose, tips, and headings in these files;
  extract the bullet list(s). A line is a source if it contains a URL; a line is
  a product if it sits under the products list. Tolerate blank lines, extra
  commentary, and trailing punctuation.

---

## First-run setup (survey the user when preferences aren't configured)

**Before the run procedure, check whether the preferences are still empty or in
their untouched template state. If so, interview the user and write their answers
into the preference files, then continue with the normal run.** This makes the
skill self-configuring on first use — the user shouldn't have to hand-edit
markdown before their first report.

### Bootstrap: create `preferences/` from `templates/` if missing

The real `preferences/` directory is **private and git-ignored**, so it is
absent in a fresh clone. Committed copies with the **same filenames** live in
`templates/`. Before the checks below, for each of `products.md`, `sources.md`,
`bug-template.md`: if `preferences/<file>` does not exist, copy
`templates/<file>` → `preferences/<file>`. This restores the exact paths the run
procedure reads, so **no other logic changes** — the AI always reads
`preferences/…`. A freshly bootstrapped `preferences/` counts as unconfigured
(it holds template placeholders), so the survey below then runs.

### Detecting an unconfigured skill

Treat a preference file as **unconfigured** if any of these hold:

- **`products.md`** has no product bullets under `## Products`, *or* the only
  bullets are the shipped placeholders (e.g. still the literal example product,
  `<product name>`, "List the product name(s)…", or similar template text).
- **`sources.md`** has no URL bullets under `## Links` *and* the section still
  only contains template prose. (Note: an intentionally-empty `sources.md` is a
  valid configured state — it triggers the web-search fallback. Only treat it as
  unconfigured if it still contains the shipped placeholder links or example
  text, not if the user deliberately cleared it. If unsure, ask.)
- **`bug-template.md`** is untouched from the shipped default. This file has a
  sensible default, so **do not force** the user to change it — just offer.

If **all** files already contain real user values, skip this section entirely and
go straight to the *Run procedure*. Never re-run the survey on a configured skill,
and never overwrite user-provided values without asking.

### Survey flow

When at least one file is unconfigured, use the `question` tool to interview the
user, then write their answers into the files. Ask only for what's missing:

1. **Products (required).** Ask which product name(s) they want to monitor (one
   or more). Then ask whether the name is ambiguous / shared with other products
   and, if so, capture a one-line scope note (what to include/exclude). Write the
   names as bullets under `## Products` and the note under `## Scope note` in
   `preferences/products.md`, preserving the file's headings and dropping the
   placeholder bullets.
2. **Sources (optional).** Ask for the URLs to scan (review pages, subreddits, X
   searches, issue trackers, etc.), one per line. Explain that leaving it empty
   is fine — the skill will fall back to an automatic web search. Write any
   provided URLs as bullets under `## Links` in `preferences/sources.md`,
   removing the placeholder links. If the user provides none and wants the
   fallback, clear the placeholder links and leave the list empty.
3. **Ticket format (optional).** Tell the user a sensible default template is
   already in place and ask if they want to customize it (fields, headers,
   priority scheme). Only edit `preferences/bug-template.md` if they ask;
   otherwise leave the default untouched.

Guidelines:

- **Preserve file structure.** Keep each file's existing headings, the human-only
  `NOTES-START`…`NOTES-END` blocks, and the helper prose; only replace the
  placeholder bullet values with the user's real answers.
- **Confirm before writing** if you're replacing anything that might be a real
  user value.
- **Write, then verify.** After saving, re-read the files and confirm they now
  parse under the *Preference-file conventions* before starting the run.
- **Then continue** straight into the *Run procedure* below using the freshly
  configured values — the survey is a prelude to the run, not a separate command.
- **Non-interactive runs** (e.g. scheduled/headless) can't survey. If preferences
  are unconfigured in that context, skip the survey, note that the skill is
  unconfigured, and rely on the web-search fallback (or stop gracefully) rather
  than blocking.

---

## Run procedure

Execute these steps in order. Do not skip the validation step (step 8).

0. **First-run check.** Before anything else, run the *First-run setup* check
   above. If the preferences are unconfigured and the run is interactive, survey
   the user and write their answers before proceeding. If already configured, skip
   straight to step 1.
1. **Load preferences.** Read `preferences/products.md`, `preferences/sources.md`,
   `preferences/bug-template.md`. Extract the product name(s) and the source URLs
   from their bullet lists (ignore prose/tips; skip `NOTES-START`…`NOTES-END`
   human-only blocks). If `products.md` has a scope note, apply it.
1b. **Fallback to web search ONLY when zero sources are configured.** This
   fallback is gated: it fires **if and only if** `sources.md` contains no URLs at
   all. **If `sources.md` has one or more URLs, the Google fallback is forbidden**
   — scan exactly those URLs with the browser (see step 4), and if one can't be
   read, record it as a source failure rather than swapping in a web search. A
   JS-heavy or login-walled configured source is *not* "no usable URLs"; it just
   means you must open it in the browser. Only when the user genuinely provided
   nothing: run a Google web search scoped to the product name(s) plus intent terms
   (`bug OR broken OR crash OR "feature request" OR review OR complaint`), treat the
   relevant top results as sources, apply the *Search scope strategy* and the
   *Reference validation gate* to every discovered link, and state clearly in the
   report that no sources were configured.
2. **Load memory.** Read `memory/reported-bugs.md`. Build an in-memory index of
   existing issues (by title, type, and normalized content) for dedupe matching.
3. **Set the run window.** Default: feedback from the last 30 days, or since the
   last run's date recorded in memory, whichever is more recent. State the window
   in the report.
3b. **Handle login-walled sources.** If a source shows a login wall (or is a
   known login-walled host like X), open it in the browser and **ask the user to
   log in, then wait for their confirmation** before scanning (see
   *Authentication*). If already logged in (persistent profile), just proceed. If
   the user skips, record a graceful source failure; do not block the run.
4. **Open each source in the browser and search it, scoped to the product** (see
   *Search scope strategy*). **Use the Playwright browser tools to load and read
   every configured source** — `browser_navigate` to open it, `browser_snapshot`/
   `browser_evaluate` to read the rendered content. `webfetch` may be tried first
   only as a quick probe for plain static pages; the moment it returns an empty
   shell, partial JS content, a login wall, or a CAPTCHA (the norm for
   marketplaces, Reddit, X, forums), **switch to the browser — do not treat the
   source as unreadable and do not fall back to Google search.** For a login-walled
   source, run the *Interactive login flow* (step 3b / *Authentication*). Only
   after the browser also genuinely fails do you record a graceful source failure.
   For each candidate, capture: source, exact URL, author, date, timestamp,
   verbatim quote, and a one-line summary. **While the browser is still open on a
   source that lacks per-item permalinks, capture a highlighted screenshot of the
   exact item** (see *Highlighted source screenshots*) — do it now, in the same
   pass, rather than re-opening the page later.
5. **Split into atomic issues, then classify** (see *One issue per ticket* and
   *Classification & emoji coding*). A single review/comment often contains
   several distinct problems or requests — split them first, then classify each.
6. **Dedupe & recurrence** against memory (see *Dedupe & upvote logic*). Split
   candidates into: NEW issues, RECURRING (echo of an existing issue), and
   already-reported-no-change (skip unless recurring).
7. **Prioritize** every NEW and RECURRING issue (see *Priority model*).
8. **Validate every reference** (see *Reference validation — mandatory gate*).
   This is non-negotiable and runs before any artifact is written.
9. **Generate artifacts:** create the per-run folder `output/<run>/` (see *Files
   this skill uses* for the naming rule), then write `output/<run>/report.md` and
   `output/<run>/tickets.md` (see *Output artifacts*). Reference any highlighted
   source screenshots saved in step 4 on each ticket's `Screenshot:` line.
10. **Update memory:** append new issues, update echo logs and priority for
    recurring issues, record the run date. (See *Dedupe & upvote logic*.)
11. **Summarize** to the user: counts by type/priority, recurring items, and any
    `⚠️ UNVERIFIED` flags that need manual attention.

---

## Source handling (derive everything from the URL)

The user only provides bare URLs. If the user provides **no** URLs at all, fall
back to an automatic Google web search (see step 1b) and treat its top results as
sources. Otherwise, for each source URL, infer how to treat it:

- **Classify the source from its host/path.**
  - A product's **own review/listing page** (e.g. a Marketplace item review tab,
    an app-store reviews page) → every item is on-scope; read reviews directly,
    newest first.
  - A **social/search page** (Reddit, X, forums, Google results) → results are
    unscoped; you must apply the *Search scope strategy* to filter to the
    product.
  - An **issue tracker** (GitHub issues, etc.) → filter by recency/labels and
    scope to the product.
- **Search URLs with an empty query.** If a URL is clearly a search page with a
  blank query (e.g. ends in `search?q=` or `q=&...`), fill in a scoped query
  built from the product name(s) plus intent terms
  (`bug OR broken OR crash OR "feature request" OR wish`).
- **Fetch strategy (browser-first for anything dynamic).** `webfetch` is only a
  quick probe for plain static pages. For X, Reddit, marketplaces, app stores,
  forums, and issue trackers — assume they are JS-rendered and **go straight to the
  browser tools** (`browser_navigate` + `browser_snapshot`/`browser_evaluate`). If
  you did try `webfetch` and it returns a login/CAPTCHA/empty shell or partial
  content, that is the signal to **escalate to the browser, not to skip the source
  or fall back to Google search.** **If the page is behind a login wall, use the
  interactive login flow** (open it, ask the user to log in, wait for confirmation
  — see *Authentication* below), then retry in the browser. Only if the browser
  *also* genuinely can't access it do you record it as a **source failure** in the
  report and move on — never fabricate content, and never silently replace a
  configured source with a Google web search.
- **Deep links.** Build a link to the exact item, inferred from the platform:
  - Reddit → the comment permalink (`.../comment/<id>/`) + `u/author` + date.
  - X/Twitter → the specific status URL + `@handle` + timestamp.
  - Review pages with no per-item anchor → record reviewer name + rating + date
    + a **highlighted** screenshot of the exact item (see *Highlighted source
    screenshots*) so it can be found instantly, plus sort order as backup.
  - Threads/forums → the specific post anchor (`#comment-<id>` / `#post-<id>`).
  See *Deep-linking rule* for the requirement this satisfies.

---

## Authentication (login-walled sources)

Some sources (X/Twitter, sometimes Reddit) block logged-out visitors. Automated
credential login does **not** work — X and similar sites detect and rate-limit
scripted logins ("your login was temporarily limited"). The reliable approach is
**interactive human login**: the skill opens the page in the browser, asks the
user to log in themselves, and waits for the user to confirm before continuing.

**A human logs in; the run never types passwords, solves CAPTCHAs, or stores
credentials.** Because the Playwright MCP browser uses a **persistent profile by
default**, once the user logs in during a run, that session stays logged in for
the rest of the run and typically persists across future runs too — so the user
usually only has to do this occasionally, not every time.

### Interactive login flow

When a source shows a login/onboarding wall (or is a known login-walled host like
`x.com`/`twitter.com`):

1. **Open the login page in the browser** with `browser_navigate` (e.g. the
   site's login URL or the source URL that redirected to login). Leave the
   browser open.
2. **Check whether already logged in first.** If the persistent profile already
   has a valid session (the feed/search renders, no login wall), skip straight to
   scanning — do not prompt the user needlessly.
3. **If not logged in, ask the user to authenticate**, then **wait**. Use the
   `question` tool (or a clear pause) with a message like: *"I've opened <site> in
   the browser. Please log in there (handle any 2FA/CAPTCHA), then choose 'Done'
   when you're logged in — or 'Skip' to skip this source."* Do not proceed until
   the user responds.
4. **On 'Done', verify** the logged-in state (navigate to the feed/search and
   confirm the wall is gone). If it still shows a wall, tell the user and offer to
   wait again once, or skip.
5. **On 'Skip' (or if the user can't log in)**, record that source as a
   **source failure** ("requires login; skipped by user") and move on — never
   fabricate content.
6. **Reuse within the run.** After a successful login, the same context is used
   for all subsequent pages on that platform in this run; do not prompt again for
   the same platform.

### Guardrails

- **Never** enter credentials, solve CAPTCHAs, or create accounts. The human does
  the login; the AI only opens the page, waits, and verifies.
- A source the user chooses not to log into is a normal, expected state →
  graceful **source failure**, not an error. Report it clearly.
- Do not read, store, or echo any credentials. There is no credentials file.
- If running fully headless / non-interactively (e.g. scheduled), interactive
  login isn't possible: rely on the persistent profile's existing session if any,
  otherwise record the source as requiring login and continue.

---

## Search scope strategy

The single biggest quality risk is scope creep — pulling feedback about the
wrong product because a name is shared across many products. Use the product
name(s) and the optional scope note from `products.md`.

**Core rule:** a candidate only counts if it clearly refers to the *specific
product* the user listed, not a different product that merely shares part of the
name.

Practical tactics:

- **AND-scope the exact product name.** Quote the full product name in queries;
  require it, not just a partial/brand word.
- **Add qualifiers** from the name itself (e.g. platform/surface words in the
  name) plus intent terms (`bug`, `broken`, `crash`, `feature request`, `wish`).
- **Apply the scope note** from `products.md` to exclude same-name products; if
  no note is given and the name is ambiguous, exclude uncertain hits and ask the
  user once if needed.
- **Prefer the product's own review/listing pages** (auto on-scope).
- **Constrain recency** to the run window.
- **When in doubt, exclude and note it.** A false positive that becomes a filed
  ticket is worse than a missed borderline mention. Log borderline drops in the
  report's "excluded / ambiguous" note.

---

## Reference validation — mandatory gate

AI hallucinates links. This gate exists to guarantee every reference in a filed
ticket is real and actually contains the quoted feedback. **No artifact is
written until this passes.**

For **every** URL attached to **every** ticket:

1. **Fetch/open it** — use the browser tools for any JS-heavy or social page (the
   norm here); `webfetch` only for plain static pages. See *Source handling* for
   the browser-first fetch strategy. An empty/walled `webfetch` result means open
   it in the browser, not mark it unverifiable.
2. **Confirm it loads** (HTTP 200 / renders, not 404/redirect-to-home/login
   wall).
3. **Confirm the quoted text actually appears** on the page (or, for a specific
   comment in a long thread, that the author + text match at that permalink).
4. **Confirm author + date** match what you recorded.

Handling failures:

- If a link cannot be verified, **do not invent or substitute another link.**
- Drop the unverifiable link and, if no valid source remains, mark the ticket
  `⚠️ UNVERIFIED` in both artifacts and list it separately for manual review.
- Never fabricate a permalink, author, timestamp, or quote. If you did not
  retrieve it from a real page, it does not go in the ticket.

State in the report how many references were checked and how many failed.

---

## Deep-linking rule (find the exact comment)

Feedback often lives inside one long thread. A link to the thread root is not
good enough — the user must be able to jump to the exact comment.

For each reference capture and record:

- **Deep permalink to the specific comment**, not the thread root. Build it from
  the platform conventions in *Source handling* (Reddit comment permalink, X
  status URL, thread post anchor, etc.).
- **Author** (username/handle/display name).
- **Date** and, where available, **timestamp** (with timezone, prefer UTC).
- If a deep link genuinely doesn't exist for that source, record enough locator
  metadata (author + date + sort order) plus a **highlighted screenshot** of the
  exact item (see *Highlighted source screenshots*) so a human can find it in
  seconds.

---

## Highlighted source screenshots (pin the exact item)

When a source has no per-item permalink (most review/listing pages — a Marketplace
review tab, an app-store page), a plain screenshot isn't enough: the reader still
has to hunt for the right review. Instead, capture a **highlighted** screenshot
that draws a box around the exact item the ticket is about, plus a label. This is
the visual equivalent of a deep link.

**Only capture a highlight when the item lacks a real deep permalink.** If a true
comment permalink exists (Reddit/X/forum), use that and skip the screenshot.

### How to capture (DOM-rect method — accurate, preferred)

Do this **in the browser**, while you already have the page open for scraping.
Never guess pixel coordinates from a static image — locate the real element and
read its exact rectangle:

1. **Find the element** for the specific review by matching the author name and a
   distinctive phrase from the quote (e.g. the review card containing both
   "Vernon Young" and its first sentence). Prefer the outermost card/container so
   the box wraps the whole item.
2. **Scroll it into view** and get its exact rect with
   `element.getBoundingClientRect()` (or Playwright's `boundingBox()`). These are
   real pixels — no guessing, no drift.
3. **Draw an overlay** at that rect: a solid red border (≈4px, `#ff2d2d`, slightly
   rounded) with a small filled label above it reading
   `SOURCE: <author> — <rating>★ <date>`. Inject it as an absolutely-positioned
   element so it sits on top of the page without altering layout.
4. **Screenshot** the relevant region (the highlighted card plus enough
   surrounding context to orient the reader) and save it into the current run's
   folder as `output/<run>/source-<slug>.png`, where `<slug>` is a short stable id
   (e.g. `vernon-young-2026-08-27`).
5. **Remove the overlay** afterward so it doesn't pollute later captures.

If the page can't be automated in the browser for some reason, fall back to a
plain (un-highlighted) screenshot and note in the ticket that the region isn't
marked — do **not** fabricate a box at guessed coordinates.

### Referencing the screenshot in the ticket

Put the saved file in the ticket's **Sources** list on the existing
`Screenshot:` line as a path relative to the run folder, e.g.
`Screenshot: source-vernon-young-2026-08-27.png` (both artifacts live in the same
`output/<run>/` folder as the screenshot), alongside the author + rating + date +
sort-order locator. One screenshot per
atomic ticket; when several tickets are split from the same review, they may
share the same highlighted screenshot of that review.

---

## One issue per ticket (atomic splitting)

**Every ticket describes exactly one issue.** One source often bundles several —
a review may list "1. add drag-and-drop 2. faster load 3. fix the diff." Each of
those becomes its **own** ticket.

Rules:

- **Split first, before classifying.** For each captured candidate, decompose it
  into the distinct problems/requests it contains (numbered lists, bullet lists,
  "also…", and separate sentences describing unrelated things are strong split
  signals).
- **One type per ticket.** If a source mixes a bug and a feature request, that is
  two tickets with two different type emojis — never a combined
  `bug + feature` ticket.
- **Prioritize each split independently.** A data-loss bug and a cosmetic request
  from the same review get different priorities.
- **Share the source across splits.** Every ticket derived from the same
  source carries that same source reference (permalink/author/date/screenshot),
  and quotes **only the portion relevant to that one issue** — not the whole
  review. Keep quotes verbatim; trim to the relevant clause.
- **Dedupe each split separately** against memory. Splitting is what lets a
  recurring single request (e.g. "drag-and-drop") accumulate echoes across many
  reviews and escalate, instead of being buried inside a multi-item ticket.
- **One memory entry per atomic issue.** Never file a memory entry that bundles
  multiple issues.

If a review is truncated and an item is only partially visible, still create the
ticket for the visible item and note the truncation in *Missing info*; do not
merge it with other items to compensate.

---

## Classification & emoji coding

Tag every issue with exactly one type emoji:

- 🐛 **Bug** — something is broken or behaves incorrectly.
- 💡 **Feature request** — user wants new capability or an enhancement.
- 💬 **General feedback** — sentiment, praise, confusion, or opinion without a
  specific actionable defect.
- 📉 **Regression** — previously worked, now broken (a bug subtype; use when the
  user indicates it used to work).

The type emoji appears in the ticket title and the report.

---

## Priority model

Assign each issue a priority. Base it on impact, then boost for signal.

Base severity:

- **P0 — Critical:** data loss, crash on launch, security/privacy issue, product
  unusable, or public reputational risk spreading fast.
- **P1 — High:** blocks a core workflow; no reasonable workaround.
- **P2 — Medium:** degrades experience; workaround exists.
- **P3 — Low:** cosmetic, minor annoyance, or nice-to-have.

Signal boost (raise priority by one level, max P0):

- **Volume:** multiple independent users reporting the same issue in the window.
- **Recurrence:** issue already exists in memory and is being reported again
  (see below) — recurring unfixed issues should climb, not stagnate.
- **Velocity:** complaints accelerating / gaining visible traction (upvotes,
  reshares).

Record the reason for any boost in the ticket.

Sort both artifacts by priority (P0 → P3), then by confidence, then by recency.

---

## Dedupe & upvote logic (memory)

Memory (`memory/reported-bugs.md`) prevents duplicate filing **and** drives
priority escalation for recurring, unfixed problems.

On each run, for every candidate:

- **No match in memory → NEW.** File a fresh ticket. After the run, append a new
  ledger entry (see the schema in `memory/reported-bugs.md`) with a stable ID,
  title, type, priority, first-seen date, status `open`, and an initial echo log
  entry.
- **Matches an existing entry → RECURRING.** Do **not** file a duplicate ticket.
  Instead:
  - Append a new line to that entry's **echo log**: date, source, author,
    permalink, quote.
  - Increment its **echo count**.
  - Re-evaluate priority with the recurrence boost; if raised, update the entry.
  - Include it in the run's `tickets.md` as an **upvote block** — a short ticket
    that references the existing issue ID and says "attach to existing bug to
    raise priority," with the new corroborating source(s). This is what the user
    pastes to upvote the existing Buganizer ticket.
- **Same-day / same-content dedupe:** never file the same content twice in one
  run. Normalize quotes (lowercase, strip punctuation/whitespace) and skip
  near-identical matches.

Matching heuristic: compare normalized title + type + core symptom. When
uncertain whether something is a match, prefer treating it as RECURRING and note
the ambiguity, rather than filing a possible duplicate.

---

## How to describe a hard-to-reproduce bug

Users rarely give clean repro steps. This section is about *technique* — how to
turn vague feedback into an actionable description. The **exact fields and layout
live in `preferences/bug-template.md`**; populate whatever repro/description
fields that template defines. Do not restate or hardcode the field list here.

Principles for filling the description, whatever the template's shape:

- **Separate their words from your interpretation.** Quote the user verbatim in
  the template's quote field; put your analysis in the description fields.
- **Distinguish observed vs. expected behavior** from what the user said.
- **Capture environment** only from what's stated or reasonably inferable, and
  **mark every inferred item as inferred** — never assert unstated specifics.
- **Never present a guess as a confirmed repro.** If the user gave steps, use
  them; if not, offer a best-guess sequence explicitly labeled as an unconfirmed
  hypothesis.
- **Name the gaps.** Explicitly list what information is still missing to
  reproduce, so the eng team knows what to ask for.

Keep it concise but complete.

---

## Output artifacts

Every run writes both files into its own per-run folder, `output/<run>/` (see
*Files this skill uses* for the naming rule), alongside any source screenshots.

### Artifact 1 — `report.md` (concise brief)

Email/doc-friendly. Keep it scannable. Include:

- **Header:** product, run date, run window, sources scanned.
- **Summary line:** totals by type (🐛/💡/💬/📉) and priority (P0–P3).
- **Top issues:** the highest-priority items, one line each with type emoji,
  priority, title, and confidence.
- **Recurring / escalating:** issues seen again this run and why priority should
  rise.
- **Sentiment trend:** are complaints about any area rising vs. the last run?
- **Validation summary:** references checked, references failed, `⚠️ UNVERIFIED`
  count.
- **Excluded / ambiguous:** borderline items dropped for scope reasons.

### Artifact 2 — `tickets.md` (copy-paste tickets)

One block per NEW issue plus one upvote block per RECURRING issue, sorted by
priority. **Each block must follow `preferences/bug-template.md` exactly**, which
is authored in clean, well-formatted markdown (bold section headers, blank lines
between sections, blockquote for the quote, bulleted sources/repro, `---` between
tickets). This file is meant to be opened and bulk copy-pasted into
Buganizer/Jira, so formatting fidelity matters.

Add a per-ticket **Confidence** tag:

- **High:** verified reference(s) + multiple independent sources.
- **Medium:** verified reference, single source.
- **Low:** single source, thin detail, or partially unverifiable.

---

## Notes on scheduling

This skill does not run itself — opencode skills are invoked, not autonomous. To
run on a schedule, use the launchd example in `scheduling/` (macOS), which calls
opencode headless with a fixed prompt like "run the product sentiment report."
See `scheduling/README.md`.
