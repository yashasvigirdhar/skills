# Competitor Pricing Tracker — Setup

## Prerequisites

- An AI coding agent with **web search** and **web fetch** capabilities (needed to discover competitors and scrape pricing pages)
- A codebase with enough context to identify what the product is (README, docs, landing page, etc.)

## Installation

1. **Copy the skill files** into your project's skills directory:

   ```bash
   # For Claude Code (adjust path for other agents)
   mkdir -p .claude/skills/competitor-pricing-tracker/references
   ```

2. **Copy these files** into the created directory:
   - `SKILL.md` → `.claude/skills/competitor-pricing-tracker/SKILL.md`
   - `references/data-model.md` → `.claude/skills/competitor-pricing-tracker/references/data-model.md`

3. **Register the skill** with your agent if required (Claude Code auto-discovers skills in `.claude/skills/`).

## First Run

No manual configuration needed — the skill bootstraps itself interactively on first run:

1. Explores your codebase to understand your product
2. Web searches for competitors in your space
3. Asks you to confirm/edit the competitor list
4. Defines feature comparison keys relevant to your industry
5. Researches pricing for each competitor
6. Writes the pricing data file and a `config.json`

To trigger the first run, invoke the skill (e.g., `/verify-competitor-pricing` in Claude Code or ask your agent to "check competitor pricing").

## What Gets Created

After the first run:

| File | Location | Purpose |
|---|---|---|
| `config.json` | Skill directory | Product info, data file path, freshness thresholds |
| `competitor-pricing-data.json` | Your docs directory (chosen during setup) | The full structured pricing database |
| `runs.log` | Skill directory | Run history for the self-improvement loop |

## Customization

- **Freshness thresholds**: Edit `config.json` → `freshnessThresholds` to change when entries are considered stale (default: 30 days) or very stale (default: 90 days)
- **Data file location**: Chosen during bootstrap. Move it later by updating `config.json` → `dataFilePath`
- **Adding competitors**: Edit the data file directly to add new platforms, or delete `config.json` and re-run the bootstrap
- **Feature keys**: Add new keys to all platform entries in the data file — the skill will pick them up on the next run
