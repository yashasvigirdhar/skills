---
name: competitor-pricing-tracker
description: "Use when you need to track competitor pricing, check if stored pricing data is stale, or do initial competitive pricing research. On first run, discovers competitors via codebase analysis and web research, then builds a comprehensive pricing database."
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - WebFetch
  - WebSearch
  - Glob
  - Grep
  - AskUserQuestion
compatibility: Requires WebSearch and WebFetch tools (or equivalent web access). No external accounts or API keys needed.
license: MIT
metadata:
  author: yashasvigirdhar
  version: "1.0"
---

# Competitor Pricing Tracker

You track and monitor competitor pricing data for the user's product. On first run, you discover the product's competitive landscape automatically — exploring the codebase, researching competitors via the web, and building a structured pricing database. On subsequent runs, you check data freshness, re-scrape stale entries, and surface pricing changes.

## Before You Start

1. Read the **Accumulated Learnings** section at the bottom of this file. Apply any lessons from previous runs.
2. Read `runs.log` in this skill's directory (if it exists) to see when pricing was last checked and what changed.
3. Read `references/data-model.md` to understand the pricing data schema.
4. Check if `config.json` exists in this skill's directory:
   - **Missing** → First Run (Bootstrap)
   - **Exists** → Regular Run

---

## First Run (Bootstrap)

When `config.json` doesn't exist, you need to discover the product and its competitors.

### Step 1: Understand the Product

Explore the codebase to figure out what this product is:

1. Read `README.md`, `CLAUDE.md`, and any docs at the repo root
2. Look for a landing page or marketing copy (common locations: `landingpage/`, `marketing/`, `website/`, `public/`, `src/pages/`)
3. Look for a pricing page or billing config (search for "pricing", "plans", "tiers", "billing" in the codebase)
4. Read `package.json`, `pyproject.toml`, or similar for project metadata

Build a mental model of:
- **Product name** and URL
- **Industry/category** (e.g., "fitness coaching software", "project management tool", "email marketing platform")
- **Target audience** (e.g., "personal trainers", "small teams", "e-commerce businesses")
- **Key features** the product offers
- **Pricing model** if visible (free tier, subscription tiers, usage-based, etc.)
- **Unit of scaling** — what the pricing scales by (clients, seats, users, contacts, projects, etc.)

Present your understanding to the user:

```
I've explored your codebase. Here's what I understand:

Product: [name] ([url])
Industry: [category]
Audience: [target]
Key features: [list]
Pricing unit: [e.g., "clients", "seats", "contacts"]

Is this accurate? Anything to correct?
```

Wait for confirmation before proceeding.

### Step 2: Discover Competitors

Using the product understanding from Step 1:

1. **Web search** for competitors using queries like:
   - "[product category] alternatives"
   - "best [product category] software [current year]"
   - "[product category] pricing comparison"
   - "top [product category] tools for [target audience]"
2. **Check the codebase** for existing competitor references:
   - Search for `competitor`, `alternative`, `comparison` in docs/
   - Look for existing competitive analysis files
3. Compile a list of 5–15 direct competitors (same category, similar audience)

Present the discovered competitors:

```
I found these competitors in your space:

 1. CompetitorA (https://...) — one-line description
 2. CompetitorB (https://...) — one-line description
 ...

Should I add or remove any?
```

Wait for the user to confirm, add, or remove competitors before proceeding.

### Step 3: Define Feature Comparison Keys

Based on the product's features and the industry, define 5–10 boolean feature keys that matter for competitive comparison. These should represent the core capabilities buyers evaluate when choosing between products.

Example for a project management tool: `taskManagement`, `timeTracking`, `ganttCharts`, `resourcePlanning`, `invoicing`, `apiAccess`, `sso`, `customFields`

Present them to the user:

```
I'll track these features across all competitors:

- featureKeyA: Description
- featureKeyB: Description
...

Want to add, remove, or rename any?
```

Wait for confirmation.

### Step 4: Research Pricing

For each confirmed competitor (and the user's own product):

1. **Find the pricing page** — web search "[competitor name] pricing"
2. **Fetch the pricing page** — use WebFetch on the pricing URL
3. **Extract structured data**:
   - Pricing model (flat tiers, per-user, usage-based, etc.)
   - All tier names, prices, and unit limits
   - Currency (note original currency if different from the default)
   - Overage rates (if applicable)
   - Add-on pricing for optional features
   - Which features are available on which tiers (feature gating)
4. **If pricing isn't publicly available** (enterprise / "contact us"), mark the price as `null` and add a note — don't guess

**When WebFetch fails** (Cloudflare block, JS-rendered page, timeout):
- Try the `www.` prefix variant of the URL
- Try WebSearch for `"[competitor] pricing [current year]"` — review articles and comparison sites often list prices
- Search for the pricing data in Google's cached version: `cache:[pricing-url]`
- Check if the page source contains pricing in `<script>` tags, JSON-LD, or embedded API responses
- If all approaches fail, note "pricing page requires browser rendering" and record whatever partial data you found. The user can fill in the rest manually or use browser automation tools on the next run.

Build the full pricing data file using the schema in `references/data-model.md`.

**Non-tier pricing models**: Not all competitors use tier-based pricing. For usage-based models (per-API-call, per-GB, per-transaction), represent each usage bracket as a "tier" where the tier name describes the usage level. For fully custom / sales-only pricing, set `tiers` to an empty array and explain the model in `pricingModel`. See `references/data-model.md` for details.

### Step 5: Save Everything

1. **Choose data file location** — ask the user where to store the pricing data file. Suggest a sensible default based on the repo structure:
   - If `docs/competitors/` exists → `docs/competitors/competitor-pricing-data.json`
   - If `docs/` exists → `docs/competitor-pricing-data.json`
   - Otherwise → `competitor-pricing-data.json` at repo root

2. **Write the pricing data file** to the agreed location. Include the user's own product in the `platforms` array for easy side-by-side comparison.

3. **Write `config.json`** in this skill's directory:
   ```json
   {
     "product": {
       "name": "Your Product",
       "url": "https://yourproduct.com",
       "industry": "your industry category"
     },
     "dataFilePath": "docs/competitors/competitor-pricing-data.json",
     "freshnessThresholds": {
       "staleDays": 30,
       "veryStaleDays": 90
     },
     "scrapingNotes": {}
   }
   ```

   The `scrapingNotes` object is used by the self-improvement loop — see **Self-Improvement** below.

4. Present a summary of all findings to the user — total competitors tracked, any that had hidden pricing, any surprises.

---

## Regular Run

When `config.json` exists, check freshness and update stale data.

### Pre-flight Check

1. Read `config.json` — verify the data file path exists and is readable
2. If the data file is missing, tell the user and ask whether to re-bootstrap or create the file from scratch

### Step 1: Load Data

1. Read `config.json` for the data file path, thresholds, and scraping notes
2. Read the pricing data file
3. Parse the freshness thresholds (`staleDays`, `veryStaleDays`)
4. Check `scrapingNotes` in config for per-competitor tips from previous runs (e.g., "use www. prefix", "pricing is in a script tag", "requires browser")

### Step 2: Report Freshness

For each platform in the data file, calculate:
- Days since `lastVerified`
- Status: **Fresh** (≤ staleDays), **Stale** (> staleDays), or **Very Stale** (> veryStaleDays)

Present the freshness report:

```
Competitor Pricing Freshness Report
====================================

| Platform       | Last Verified | Days Ago | Status     |
|----------------|---------------|----------|------------|
| CompetitorA    | 2026-03-05    | 21       | Fresh      |
| CompetitorB    | 2026-01-15    | 70       | Stale      |
| ...            | ...           | ...      | ...        |

Fresh: N | Stale: N | Very Stale: N
```

If everything is fresh, report that and stop (no re-scraping needed).

### Step 3: Update Stale Entries

For each stale or very stale platform:

1. Check `config.json` → `scrapingNotes` for this platform's ID — if a previous run recorded a working approach (e.g., "use www. prefix", "check script tags"), use that approach first
2. WebFetch their `pricingUrl` (or use the known working approach)
3. Apply the same fallback chain as in the bootstrap if the fetch fails
4. Extract current pricing and compare with stored data
5. Report differences clearly:

```
### CompetitorB (stale — 70 days)
- Stored: Starter $19/mo, Pro $49/mo
- Current: Starter $19/mo, Pro $59/mo  ← PRICE CHANGED (+$10)
- New tier discovered: Enterprise $149/mo
- Feature change: AI features now included on Pro (was add-on)
```

### Step 4: Apply Updates

Ask the user if they want to update the data file with the new findings.

If yes:
- Update the relevant platform entries in the JSON
- Set `lastVerified` to today's date for each updated platform
- Update `meta.lastFullAudit` if all platforms were checked in this run
- Preserve any fields the user has manually added (e.g., `highlightNote`)

---

## After the Run

1. **Append to `runs.log`** in this skill's directory:
   ```
   ## YYYY-MM-DD
   - Run type: bootstrap / regular
   - Platforms checked: [count]
   - Stale: [count], Very stale: [count]
   - Changes found: [brief list or "none"]
   - Data file updated: yes/no
   - Scraping issues: [list platforms that were hard to scrape and why, or "none"]
   ```

2. Run the **Self-Improvement** steps below.

## Self-Improvement

After every run, review what happened and update this skill to make the next run smoother. This is how the skill gets smarter over time.

### 1. Update Scraping Notes

For each platform you scraped during this run, check if there was anything noteworthy about the scraping approach:

- Did you need a fallback strategy? (e.g., `www.` prefix, Google cache, script tag extraction)
- Did the pricing page structure change since last time?
- Did a previously-working approach stop working?
- Was the page JS-rendered and WebFetch couldn't get the pricing?

Record what worked (or what failed) in `config.json` → `scrapingNotes` keyed by platform ID:

```json
{
  "scrapingNotes": {
    "competitorA": "Pricing is in a <script type=\"application/ld+json\"> tag — WebFetch works but parse the JSON-LD, not the visible HTML",
    "competitorB": "Must use www.competitorb.com — bare domain redirects to a Cloudflare challenge page",
    "competitorC": "JS-rendered — WebFetch returns empty pricing section. Last worked via WebSearch fallback (2026-03-26)"
  }
}
```

This way, the next run can skip straight to the approach that works instead of rediscovering it.

### 2. Update Accumulated Learnings

Review this run for **generalizable** lessons — things that would help with ANY competitor, not just a specific one (specific ones go in `scrapingNotes`):

- New scraping pattern that worked broadly (add it)
- A learning that's no longer relevant because the underlying issue was fixed (remove it)
- A pricing model pattern you hadn't seen before that needed special handling (add how you handled it)

### 3. Maintain Competitor List

If during this run you discovered that:
- A competitor was **acquired or shut down** → ask the user if they want to remove it from the data file and config
- A **new competitor** appeared in search results or was mentioned on comparison sites → ask the user if they want to add it
- A competitor **rebranded or changed domains** → update the platform entry's `url` and `pricingUrl`

### 4. Validate Data Model Consistency

Check the data file for consistency issues:
- Are all platforms using the same set of feature keys? If a new feature was added to one platform, it should be added (as `true` or `false`) to all others.
- Are any `lastVerified` dates in the future? (indicates a bug in a previous run)
- Are there platforms in the data file that aren't in the competitor list, or vice versa?

Fix any issues found and note them in the runs.log.

---

## Accumulated Learnings

_This section is maintained by the agent after each run. Each entry should include the date and why it matters._

### Pre-seeded: Pricing page scraping tips

- **JavaScript-rendered pricing pages**: WebFetch retrieves raw HTML. If a pricing page shows "Loading..." or empty containers, prices are rendered client-side. Try searching for pricing data in `<script>` tags, JSON-LD, or API responses embedded in the page source. If that fails, try WebSearch as a secondary source — review articles often list the exact prices. Record what works in `scrapingNotes` so the next run doesn't re-discover this.
- **"Contact us" / enterprise tiers**: Many competitors hide enterprise pricing. Don't guess — mark the price as `null` with a note. The user can fill it in manually from sales conversations or public case studies.
- **Currency detection**: Some sites detect visitor location and show localized prices. If prices appear in an unexpected currency, check for a locale parameter in the URL (e.g., `?currency=USD`) or a currency switcher on the page.
- **Annual vs. monthly pricing**: Many pricing pages default to showing annual pricing (lower monthly rate). Always check for a monthly/annual toggle and capture the **monthly** price. Note the annual discount separately if relevant.
- **A/B tested pricing**: Competitors occasionally A/B test pricing pages. If fetched prices differ from a previous run by a small amount, flag it as a potential A/B test rather than a confirmed change.
- **Freemium tiers**: Always check for free tiers or free trials — they're sometimes hidden below the paid plans or listed on a separate page. These matter for competitive positioning.
- **WebFetch fallback chain**: When the primary fetch fails, try in order: (1) `www.` prefix variant, (2) WebSearch for "[competitor] pricing [year]", (3) Google cache (`cache:[url]`), (4) check `<script>` tags in raw HTML for embedded data. Record which approach works in `scrapingNotes`.
