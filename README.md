# Agent Skills

Reusable [Agent Skills](https://agentskills.io) — portable instructions that give AI agents new capabilities and expertise.

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
| [`nightly-qa`](./nightly-qa/) | Automated E2E browser testing via Chrome DevTools MCP. Writes checkpoint reports, fixes bugs, opens PRs, and learns from each run. |

## How It Works

Each skill directory follows the [Agent Skills specification](https://agentskills.io/specification):

```
nightly-qa/
├── SKILL.md                # Skill template — framework + example test flows
├── SETUP.md                # Agent-readable setup instructions
└── references/             # Supplementary docs loaded on demand
    └── chrome-devtools-gotchas.md
```

- **`SETUP.md`** — Point your agent here. It reads your codebase, asks targeted questions, and generates a customized SKILL.md for your project.
- **`SKILL.md`** — The generalized skill template. Contains the framework (checkpointing, change analysis, self-improvement) with example test flows. The setup agent uses this as the base and fills in your project-specific details.
- **`references/`** — Detailed reference material loaded on demand during runs ([progressive disclosure](https://agentskills.io/what-are-skills#how-skills-work)).

After setup, the installed skill can be invoked on demand or scheduled to run automatically (if your agent supports it). It improves itself after each run — learning from failures, adapting to code changes, and growing its test coverage over time.

## Contributing

Found a useful pattern? Open a PR. Keep skills generic — project-specific details belong in your local installed copy, not here.
