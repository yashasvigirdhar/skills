---
name: competitor-backlink-audit
description: "Use when the user wants to audit competitor backlink profiles using the free Ahrefs tool to identify link-building opportunities, or when setting up competitive SEO analysis."
allowed-tools:
  - mcp__chrome-devtools-real__*
  - mcp__chrome-devtools-isolated__*
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
compatibility: Requires Chrome DevTools MCP server, a browser with remote debugging enabled, and git. No Ahrefs account needed — uses the free Backlink Checker tool.
license: MIT
metadata:
  author: yashasvigirdhar
  version: "1.0"
---

# Competitor Backlink Audit via Ahrefs

You run competitive backlink audits using the free Ahrefs Backlink Checker via Chrome DevTools MCP. You navigate a real browser to Ahrefs, extract backlink data for each competitor, and compile a structured report with actionable link-building opportunities.

## Before You Start

1. Read the **Accumulated Learnings** section at the bottom of this file.
2. Read [references/ahrefs-gotchas.md](references/ahrefs-gotchas.md) for known Ahrefs free tool pitfalls.
3. Read `runs.log` in this skill's directory (if it exists) to see previous run history.
4. Check if `config.json` exists in this skill's directory.
   - **If it exists**: read it for the competitor list and output paths. Proceed to **Pre-flight Checks**.
   - **If it does NOT exist**: this is a first run. Proceed to **First Run Bootstrap**.

## First Run Bootstrap

When no `config.json` exists, you need to discover the product and its competitors before running the audit.

### Step 1: Understand the Product

Explore the codebase to understand what the product does:
- Read the README, CLAUDE.md, or any product overview docs
- Check the landing page or marketing copy if available
- Identify the product category, target audience, and key differentiators

Summarize your understanding to the user and confirm you've got it right.

### Step 2: Identify Competitors

Based on the product category:
1. Use web search to find direct competitors (same product category, same target audience)
2. Look for competitor mentions in the codebase (docs, marketing copy, config files)
3. Aim for 8-12 competitors spanning:
   - **Direct competitors**: Same product, same audience
   - **Adjacent competitors**: Similar product, overlapping audience
   - **Market leaders**: Dominant players in the broader space (useful for backlink discovery even if not direct competitors)

Present the competitor list to the user. Ask them to confirm, add, or remove entries.

### Step 3: Ask About Output Location

Ask the user where audit reports should be saved. Suggest a sensible default based on the project structure (e.g., `docs/seo/backlink-audit/` if a `docs/` directory exists, or `backlink-audit/` at the project root otherwise).

### Step 4: Create config.json

After the user confirms the competitor list and output location, create `config.json` in this skill's directory:

```json
{
  "competitors": [
    { "name": "CompanyName", "domain": "example.com" }
  ],
  "ahrefs_url_template": "https://ahrefs.com/backlink-checker/?input={domain}&mode=subdomains",
  "data_files": {
    "raw_data": "<output_dir>/raw-data.md",
    "report": "<output_dir>/report.md"
  }
}
```

Also create the output directory:
```bash
mkdir -p <PROJECT_ROOT>/<output_dir>
```

Then proceed to **Pre-flight Checks**.

## Pre-flight Checks

1. **Chrome DevTools MCP** must be connected. Verify by calling `list_pages` — if it fails, tell the user to open Chrome with remote debugging enabled.
2. **Read `config.json`** for the competitor list and output paths.

## Procedure

### Step 1: Navigate to Ahrefs

For each competitor from `config.json`, navigate directly to the results URL using the `ahrefs_url_template` (replace `{domain}` with the competitor's domain).

Use the Chrome DevTools `navigate_page` tool with the full URL. This is significantly faster than filling the search form manually.

### Step 2: Wait for Results

Use `wait_for` with text `["Backlink profile for DOMAIN_NAME"]` and a 15-second timeout.

Results appear in a **dialog modal overlay** on the page.

If the wait times out, check for Cloudflare challenge — see [references/ahrefs-gotchas.md](references/ahrefs-gotchas.md) for handling.

### Step 3: Extract Data

The results dialog contains:
- **Header metrics**: DR (Domain Rating), backlink count (with dofollow %), linking websites count (with dofollow %)
- **Backlink table**: ~20 rows, one per referring domain — shows referring domain DR, page title, URL, link type

Extract using one of:
- **Preferred**: `take_snapshot` for the full accessibility tree
- **Fallback**: `evaluate_script` with `document.querySelectorAll` targeting the dialog content — use this when the snapshot is too large or truncated

### Step 4: Record Data

Append each competitor's data to the raw data file (path from `config.json` → `data_files.raw_data`) in this format:

```markdown
## N. CompetitorName (domain.com)

**DR: XX** | Backlinks: XXK (XX% dofollow) | Linking websites: XXK (XX% dofollow)

| DR | Referring Domain | Page Title | Link Type |
|----|-----------------|------------|-----------|
| XX | domain.com | Page Title | Type description |
...

**Key insights:**
- Bullet points summarizing notable patterns
```

### Step 5: Pace Yourself (Cloudflare Mitigation)

Cloudflare Turnstile bot detection activates after several sequential queries. See [references/ahrefs-gotchas.md](references/ahrefs-gotchas.md) for detailed mitigation strategies.

Quick rules:
- After each competitor, the natural analysis/recording time usually provides enough pause.
- After 4-5 competitors, take a longer pause (2-3 minutes) or close and reopen the Ahrefs tab.
- Record data immediately after each competitor — don't batch.

### Step 6: Handle Free Tool Limitations

Some domains trigger free tool limitations:
- **"No backlinks in our index"**: DR is still shown. Record the DR and note the limitation. Try the `www.` prefix (e.g., `www.domain.com` instead of `domain.com`) — the free tool treats these as separate targets and one may return full data.
- **"Backlinks cannot be shown because the domain is too large"**: Aggregate numbers may still be shown. Record what's available, then retry with `www.` prefix — this often bypasses the limitation.

## Post-Run

After completing the audit for all competitors:

1. **Generate/update the report** (path from `config.json` → `data_files.report`):
   - Executive summary table (all competitors, DR, backlinks, linking sites)
   - Common referring domains (sites linking to multiple competitors — highest-value targets)
   - High-DR referring domains (DR 50+) sorted by DR
   - Link type distribution across competitors
   - Actionable recommendations: which sites to target, what kind of content/outreach would work

2. **Append to `runs.log`** in this skill's directory:
   ```
   ## YYYY-MM-DD
   - Competitors audited: [count] ([list names])
   - Cloudflare challenges: [count, or "none"]
   - Free tool limitations hit: [list domains that returned no data or "too large", or "none"]
   - Report: [path to report file]
   ```

3. **Update Accumulated Learnings**: Review what happened during this run. If any of the following occurred, add a dated entry to the **Accumulated Learnings** section below:
   - A new Cloudflare mitigation strategy that worked (or one that stopped working)
   - A new Ahrefs UI change that affected extraction (selectors, dialog structure, etc.)
   - A competitor domain that needed special handling (and what worked)
   - A technique that made extraction faster or more reliable
   - Anything that would save time or avoid mistakes on the next run

   Do NOT add entries that duplicate what's already in [references/ahrefs-gotchas.md](references/ahrefs-gotchas.md) — only add genuinely new knowledge.

4. **Update config.json if needed**: If a competitor's domain changed, a company was acquired, or a new relevant competitor was discovered during the run, update `config.json` to keep it current.

## Accumulated Learnings

_This section is maintained by the agent after each run. It captures reusable knowledge that improves future runs. Each entry should include the date and why it matters._
