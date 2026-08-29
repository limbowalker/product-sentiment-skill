# Sources

Paste one link per line. That's it. Everything below the marker is a bullet list
of URLs the AI will search/scan for product feedback.

The AI figures out the rest per URL (how to search it, how to build deep links,
whether it needs a browser) — you don't configure any of that here.

The AI reads these pages in a real browser (they're JavaScript-rendered and/or
login-walled — a plain fetch won't work), so a browser is required to run this
skill against real sources.

If you leave the list COMPLETELY empty, and only then, the AI falls back to an
automatic Google web search for your product(s) and uses the top results as
sources (noted in the report). As long as you list any URL here, the AI scans
exactly those in the browser and never silently swaps in a web search — an
unreachable source is reported as a source failure instead.

Tips (optional):

- A product's own review page (e.g. a Marketplace review tab) gives the cleanest,
  fully on-scope results.
- Social sites (Reddit, X) may be blocked without login. When that happens the
  AI opens the site in the browser and asks you to log in, then waits for you to
  confirm before continuing. Your login persists across runs (persistent browser
  profile), so you usually only do this occasionally. If you skip it, the AI just
  reports that source as unreachable.

<!-- NOTES-START (ignored by the AI) — your private reminders go here. NOTES-END -->

## Links

- https://example.com/your-product-review-page
- https://www.reddit.com/search/?q=your+product
