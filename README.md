# Claude Code Skills

Reusable [Claude Code](https://claude.ai/code) scheduled skills that can be adapted to any project.

## What are Claude Code Skills?

Claude Code supports [scheduled tasks](https://docs.anthropic.com/en/docs/claude-code/scheduled-tasks) — autonomous agents that run on a cron schedule. Each task is defined by a `SKILL.md` file that tells Claude what to do, how to verify its work, and how to improve over time.

This repo contains generalized versions of skills that have been battle-tested on real projects.

## Available Skills

### [`nightly-qa`](./nightly-qa/SKILL.md)

Automated E2E QA testing using Chrome DevTools MCP. Runs nightly against your local dev environment.

**What it does:**
- Runs through your app's critical flows in a real browser via Chrome DevTools Protocol
- Takes screenshots on failure, checks console errors and network requests
- Writes a detailed report with pass/fail status for every test step
- If it finds a bug, it fixes the code, creates a branch, and opens a PR
- Learns from each run — maintains an "Accumulated Learnings" section that grows over time
- Adapts to code changes — analyzes git diffs to add ad-hoc checks for new functionality

**Key design decisions:**
- **Mandatory checkpointing** — results are written to the report file after every single step, so nothing is lost if context compresses mid-run
- **Self-improving** — the skill file itself is updated after each run with new learnings and test adjustments
- **Change-aware** — diffs since the last run are analyzed to focus testing on what changed
- **Non-destructive** — all test data is prefixed (e.g., `[QA-NIGHTLY]`) and cleaned up after every run

## How to Use

### 1. Copy the skill to your Claude Code config

```bash
mkdir -p ~/.claude/scheduled-tasks/nightly-qa
cp nightly-qa/SKILL.md ~/.claude/scheduled-tasks/nightly-qa/SKILL.md
```

### 2. Customize for your project

Open the `SKILL.md` and:

1. **Fill in the Configuration table** at the top — set your project root, ports, URLs, auth commands, etc.
2. **Replace the example test flows** with your actual app's critical paths. The examples show the structure; your tests should cover your signup, core CRUD, error handling, and key pages.
3. **Update the cleanup section** with your database schema's FK cascade order.
4. **Update auth setup** with your project's session/cookie mechanism.

### 3. Schedule it

```bash
# Schedule to run nightly at 10 PM
claude schedule add --name nightly-qa --cron "0 22 * * *" --skill ~/.claude/scheduled-tasks/nightly-qa/SKILL.md
```

Or run it manually:
```bash
claude run --skill ~/.claude/scheduled-tasks/nightly-qa/SKILL.md
```

### 4. Let it improve itself

After each run, the skill updates its own `SKILL.md` with learnings. Over time, it becomes increasingly reliable and comprehensive for your specific app.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- [Chrome DevTools MCP server](https://github.com/anthropics/claude-code/tree/main/packages/chrome-devtools-mcp) configured in your `.mcp.json`
- Local dev servers running (frontend + backend)
- A database accessible for test data cleanup (PostgreSQL recommended)

## Writing Good Test Flows

Tips from running this across hundreds of nightly runs:

1. **Prefix all test data** — use a consistent prefix like `[QA-NIGHTLY]` so cleanup queries are reliable
2. **Use non-routable email domains** — `test.local`, `example.com`, etc. Never use real email domains in test data
3. **Respect FK order in cleanup** — delete child records before parents, or cleanup will fail silently in batch mode
4. **Don't modify demo/seed data** — create your own test records, verify read-only on existing data
5. **Keep phases independent** — if Phase 3 fails, Phase 4 should still be runnable
6. **Verify both happy and sad paths** — empty fields, invalid IDs, expired sessions
7. **Check console AND network** — a page can render "correctly" while silently failing API calls

## Contributing

Found a useful pattern? Open a PR. Keep skills generic — project-specific details belong in your local copy, not here.
