# Agent Skills

Reusable [Agent Skills](https://agentskills.io) — portable instructions that give AI agents new capabilities and expertise. These are skills I use in my daily workflow, vetted over several weeks of real usage before being generalized and shared here.

Works with any skills-compatible agent: Claude Code, Cursor, Copilot, Gemini CLI, Goose, Roo Code, Amp, and [others](https://agentskills.io/home).

## Setup

Each skill has a `SETUP.md` that your agent can follow to configure and install the skill for your project. Point your agent to it:

```
Read skills/nightly-qa/SETUP.md and follow the setup instructions for my project.
```

The agent will explore your codebase, ask you a few targeted questions, generate a project-specific skill file, install it, and optionally do a first test run.

## Available Skills

| Skill | What it does |
|---|---|
| [`nightly-qa`](./nightly-qa/) | Human-like E2E browser testing that gets smarter with each run. Opens a real browser, clicks through your app's flows, checks for errors — agnostic of your tech stack. Analyzes git diffs to discover new changes, writes checkpoint reports, fixes bugs it discovers, and opens PRs for them. |
| [`competitor-backlink-audit`](./competitor-backlink-audit/) | Competitive SEO backlink analysis using the free Ahrefs Backlink Checker via a real browser. On first run, discovers your competitors automatically. Extracts DR, backlink profiles, and linking domains for each competitor, then generates an actionable report with link-building opportunities. No Ahrefs account needed. |
| [`competitor-pricing-tracker`](./competitor-pricing-tracker/) | Track and monitor competitor pricing with automatic staleness detection. On first run, explores your codebase to understand the product, discovers competitors via web research, confirms them interactively, then builds a structured pricing database with tiers, add-ons, feature gating, and overage rates. Subsequent runs re-scrape stale entries and surface changes. |
| [`feature-inventory`](./feature-inventory/) | Create and maintain a YAML feature inventory — a single source of truth cataloging every user-facing capability (frontend routes, API endpoints, tests, docs, feature flags). On first run, explores the codebase, detects the stack, proposes a schema and initial entries, and presents a single confirmation gate. Subsequent runs detect drift between inventory and code, auto-commit fixes, or open a PR if `gh` is available. |

## How It Works

Each skill directory follows the [Agent Skills specification](https://agentskills.io/specification):

```
skill-name/
├── SKILL.md                # Skill template — the core instructions
├── SETUP.md                # Agent-readable setup instructions
└── references/             # Supplementary docs loaded on demand
```

- **`SETUP.md`** — Point your agent here. It reads your codebase, asks targeted questions, and generates a customized SKILL.md for your project.
- **`SKILL.md`** — The generalized skill template. Contains the framework and workflow with example content. The setup agent uses this as the base and fills in your project-specific details.
- **`references/`** — Detailed reference material loaded on demand during runs ([progressive disclosure](https://agentskills.io/what-are-skills#how-skills-work)).
- **`config.json`** — Created on first run (not during setup). The skill explores your project, asks you to confirm its findings, and writes the config. This keeps setup fast and config accurate.

After setup, the installed skill can be invoked on demand or run as a recurring task via cron, Claude Code scheduled tasks, or any similar scheduling mechanism your agent supports. It improves itself after each run — learning from failures, adapting to code changes, and growing its test coverage over time.

## Contributing

Found a useful pattern? Open a PR. Keep skills generic — project-specific details belong in your local installed copy, not here.
