# Competitor Backlink Audit — Setup Guide

You are setting up a competitor backlink audit skill for the user's project. This skill uses the free Ahrefs Backlink Checker via Chrome DevTools MCP to analyze competitor backlink profiles and identify link-building opportunities.

Follow these steps in order.

## Step 1: Prerequisites Check

1. **Chrome DevTools MCP** — check if the project or user has a Chrome DevTools MCP server configured (look for `chrome-devtools` in `.mcp.json`, MCP config files, or equivalent)
   - If not found: tell the user they need to set up Chrome DevTools MCP and stop here until resolved
   - Either `chrome-devtools-real` (connected to user's actual Chrome) or `chrome-devtools-isolated` (standalone instance) works
2. **Git repo** — verify the project directory is a git repository

## Step 2: Install the Skill

Save the SKILL.md and `references/` directory to the project's skill directory. Common locations:
- `<PROJECT_ROOT>/.claude/skills/competitor-backlink-audit/` (Claude Code)
- The project's `.agent/skills/` directory
- Wherever the user specifies

```bash
mkdir -p <install_path>/references
cp SKILL.md <install_path>/SKILL.md
cp references/ahrefs-gotchas.md <install_path>/references/ahrefs-gotchas.md
```

**Do NOT create config.json** — the skill creates it on first run after exploring the product and researching competitors. This ensures the competitor list is informed by actual research, not guesses.

## Step 3: First Run (Optional)

Ask the user: "Want me to do a first run now? I'll explore your product, research competitors, and run the full audit."

If they want a first run, execute the installed SKILL.md. The First Run Bootstrap section will handle:
1. Understanding the product from the codebase
2. Researching and confirming competitors with the user
3. Asking where to save reports
4. Creating `config.json`
5. Running the audit
