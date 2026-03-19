# Claude Code Skills

Reusable [Claude Code](https://claude.ai/code) scheduled skills — automated agents that run on a cron and improve themselves over time.

## Setup

Each skill has a `SETUP.md` that an AI agent can follow to configure and install the skill for your project. Point your agent to it:

```
Read skills/nightly-qa/SETUP.md and follow the setup instructions for my project.
```

The agent will explore your codebase, ask you a few targeted questions, generate a project-specific skill file, install it, and optionally do a first test run.

## Available Skills

| Skill | What it does |
|---|---|
| [`nightly-qa`](./nightly-qa/) | Automated E2E browser testing via Chrome DevTools MCP. Runs nightly, writes reports, fixes bugs, opens PRs, and learns from each run. |

## How It Works

Each skill directory contains:

- **`SETUP.md`** — Agent-readable setup instructions. The agent reads your codebase, asks targeted questions, and generates a customized skill file for your project.
- **`SKILL.md`** — The generalized skill template. Contains the framework (checkpointing, change analysis, self-improvement) with example test flows. The setup agent uses this as the base and fills in your project-specific details.

After setup, the installed skill lives at `~/.claude/scheduled-tasks/<skill-name>/SKILL.md` and runs autonomously on your chosen schedule.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- Chrome DevTools MCP server configured in `.mcp.json`
- Local dev servers (frontend + backend)

## Contributing

Found a useful pattern? Open a PR. Keep skills generic — project-specific details belong in your local installed copy, not here.
