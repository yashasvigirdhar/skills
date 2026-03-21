# Ahrefs Free Backlink Checker — Gotchas

Common pitfalls when using the free Ahrefs Backlink Checker via Chrome DevTools MCP. Discovered across multiple audit runs.

## Cloudflare Turnstile

Ahrefs uses Cloudflare Turnstile bot detection. It activates after several sequential queries.

**Mitigation strategies (in order of preference):**

1. **Natural pacing**: The time spent analyzing and recording data after each competitor usually provides enough delay. Don't rush through competitors — the recording step is your natural cooldown.
2. **Batch breaks**: After 4-5 competitors, take a longer pause (2-3 minutes) or close and reopen the Ahrefs tab.
3. **"Verify you are human" checkbox**: Click the checkbox inside the Turnstile iframe, wait 3-5 seconds.
4. **Full-page challenge** (page title "Just a moment..."): Click the checkbox, then reload the page and re-navigate to the Ahrefs URL.
5. **Persistent challenges**: Close the tab entirely, open a new one, and navigate fresh.

**Key insight**: Record data immediately after each competitor. If Cloudflare interrupts your flow mid-batch, you don't lose any data.

## The www. Prefix Trick

The free tool treats `domain.com` and `www.domain.com` as separate targets. When one returns no results or triggers a "too large" limitation, the other often works:

- **"No backlinks in our index"** for `domain.com` → try `www.domain.com`
- **"Backlinks cannot be shown because the domain is too large"** for `domain.com` → `www.domain.com` often bypasses this limit

Always try the `www.` prefix before recording a competitor as "no data available."

## Free Tool Limitations

- **~20 backlinks shown**: The free tool shows only the top ~20 referring domains. Paid Ahrefs shows all. For competitive analysis this is sufficient — the top 20 by DR are the most actionable targets anyway.
- **"Domain too large"**: Very large sites (DR 70+) sometimes trigger this. Aggregate metrics (total backlinks, linking websites) are still shown. The `www.` prefix trick often reveals the individual backlinks.
- **Rate limiting**: No explicit rate limit, but Cloudflare Turnstile enforces implicit throttling. See section above.

## Data Extraction

- **Preferred method**: `take_snapshot` captures the full accessibility tree of the results dialog. Fast and reliable for most competitors.
- **Fallback**: When the snapshot is too large or truncated (happens with very backlink-rich competitors), use `evaluate_script` with `document.querySelectorAll` targeting the dialog content. This gives you programmatic access to the DOM.
- **Dialog structure**: Results always appear in a modal overlay. The structure is consistent: header metrics section + backlink table. Parsing logic doesn't change between competitors.

## URL-based Navigation

Navigate directly to `https://ahrefs.com/backlink-checker/?input=DOMAIN&mode=subdomains` instead of filling the search form. This is significantly faster and avoids potential issues with form interaction.
