# Competitor Pricing Tracker — Setup Guide

You are setting up a competitor pricing tracker skill for the user's project. This skill discovers competitors via web research, builds a structured pricing database, and keeps it up to date by re-scraping stale entries on subsequent runs.

Follow these steps in order.

## Step 1: Prerequisites Check

1. **Web search + web fetch** — verify you have access to `WebSearch` and `WebFetch` tools (or equivalent). These are required to discover competitors and scrape pricing pages.
   - If not available: tell the user this skill requires web access and stop here until resolved
2. **Git repo** — verify the project directory is a git repository

## Step 2: Install the Skill

Save the SKILL.md and `references/` directory to the project's skill directory. Common locations:
- `<PROJECT_ROOT>/.claude/skills/competitor-pricing-tracker/` (Claude Code)
- The project's `.agent/skills/` directory
- Wherever the user specifies

```bash
mkdir -p <install_path>/references
cp SKILL.md <install_path>/SKILL.md
cp references/data-model.md <install_path>/references/data-model.md
```

**Do NOT create config.json** — the skill creates it on first run after exploring the codebase, researching competitors, and confirming them with the user. This ensures the competitor list is informed by actual research, not guesses.

## Step 3: First Run (Optional)

Ask the user: "Want me to do a first run now? I'll explore your product, research competitors, and build the pricing database."

If they want a first run, execute the installed SKILL.md. The First Run Bootstrap section will handle:
1. Understanding the product from the codebase
2. Discovering and confirming competitors with the user
3. Defining feature comparison keys for the industry
4. Researching pricing for each competitor
5. Choosing where to store the data file
6. Creating `config.json` and the pricing database
