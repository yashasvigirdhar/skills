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
compatibility: claude-code, cursor, goose, roo-code, amp, gemini-cli, copilot
license: MIT
metadata:
  author: yashasvigirdhar
---

# Competitor Pricing Tracker

Track, monitor, and keep competitor pricing data accurate. On first run, discovers your product's competitive landscape automatically — explores your codebase, researches competitors via the web, and builds a structured pricing database. On subsequent runs, checks data freshness and re-scrapes stale entries.

## Before You Start

1. Read the **Accumulated Learnings** section at the bottom of this file.
2. Read `runs.log` in this skill's directory (if it exists) to see when pricing was last checked.
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

Present your understanding to the user:

```
I've explored your codebase. Here's what I understand:

Product: [name] ([url])
Industry: [category]
Audience: [target]
Key features: [list]

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

For each confirmed competitor (and your own product):

1. **Find the pricing page** — web search "[competitor name] pricing"
2. **Fetch the pricing page** — use WebFetch on the pricing URL
3. **Extract structured data**:
   - Pricing model (flat tiers, per-user, usage-based, etc.)
   - All tier names, prices, and user/seat/client limits
   - Currency (note original currency if different from your default)
   - Overage rates (if applicable)
   - Add-on pricing for optional features
   - Which features are available on which tiers (feature gating)
4. **If pricing isn't publicly available** (enterprise / "contact us"), mark the price as `null` and add a note — don't guess

Build the full pricing data file using the schema in `references/data-model.md`.

### Step 5: Choose Data File Location

Ask the user where to store the pricing data file. Suggest a sensible default based on the repo structure:
- If `docs/competitors/` exists → `docs/competitors/competitor-pricing-data.json`
- If `docs/` exists → `docs/competitor-pricing-data.json`
- Otherwise → `competitor-pricing-data.json` at repo root

### Step 6: Save Everything

1. **Write the pricing data file** to the agreed location. Include your own product in the `platforms` array for easy side-by-side comparison.

2. **Write `config.json`** in this skill's directory:
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
     }
   }
   ```

3. Present a summary of all findings to the user — total competitors tracked, any that had hidden pricing, any surprises.

---

## Regular Run

When `config.json` exists, check freshness and update stale data.

### Step 1: Load Data

1. Read `config.json` for the data file path and thresholds
2. Read the pricing data file
3. Parse the freshness thresholds (`staleDays`, `veryStaleDays`)

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

1. Web-search / WebFetch their `pricingUrl`
2. Extract current pricing using the same approach as the bootstrap
3. Compare fetched pricing with stored data
4. Report differences clearly:

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
   ```

2. If any learning is **generalizable** and would help future runs, add it to the **Accumulated Learnings** section below.
3. Remove any accumulated learnings that are no longer relevant.

## Accumulated Learnings

_This section is maintained by the agent after each run. Each entry should include the date and why it matters._

### Pre-seeded: Pricing page scraping tips

- **JavaScript-rendered pricing pages**: WebFetch retrieves raw HTML. If a pricing page shows "Loading..." or empty containers, prices are rendered client-side. Try searching for pricing data in `<script>` tags, JSON-LD, or API responses embedded in the page source. If that fails, note "requires browser rendering" and suggest using browser automation tools if available.
- **"Contact us" / enterprise tiers**: Many competitors hide enterprise pricing. Don't guess — mark the price as `null` with a note. The user can fill it in manually from sales conversations or public case studies.
- **Currency detection**: Some sites detect visitor location and show localized prices. If prices appear in an unexpected currency, check for a locale parameter in the URL (e.g., `?currency=USD`) or a currency switcher on the page.
- **Annual vs. monthly pricing**: Many pricing pages default to showing annual pricing (lower monthly rate). Always check for a monthly/annual toggle and capture the **monthly** price. Note the annual discount separately if relevant.
- **A/B tested pricing**: Competitors occasionally A/B test pricing pages. If fetched prices differ from a previous run by a small amount, flag it as a potential A/B test rather than a confirmed change.
- **Freemium tiers**: Always check for free tiers or free trials — they're sometimes hidden below the paid plans or listed on a separate page. These matter for competitive positioning.
