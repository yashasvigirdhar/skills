# Feature Inventory — Setup Guide

You are setting up a feature inventory skill for the user's project. This skill creates and maintains a YAML feature inventory — a single source of truth cataloging every user-facing capability (frontend routes, API endpoints, tests, docs, feature flags) in one structured list.

Follow these steps in order.

## Step 1: Prerequisites Check

1. **Git repo** — verify the project directory is a git repository. Not strictly required for the skill to run, but `auto_commit` and the PR flow during drift-sync need it.
2. **`gh` CLI (optional)** — if the user wants the `auto_commit: false` + PR flow, verify the `gh` CLI is installed and authenticated. Not required otherwise.
3. **YAML parser** — no external dependency needed; the agent reads and writes YAML directly.

No API keys, external accounts, or special tooling required.

## Step 2: Install the Skill

Save `SKILL.md` and the `references/` directory to the project's skill directory. Common locations:

- `<PROJECT_ROOT>/.claude/skills/feature-inventory/` (Claude Code)
- The project's `.agent/skills/` directory
- Wherever the user's agent looks for skills

```bash
mkdir -p <install_path>/references
cp SKILL.md <install_path>/SKILL.md
cp references/schema.md <install_path>/references/schema.md
cp references/discovery-patterns.md <install_path>/references/discovery-patterns.md
cp references/philosophy.md <install_path>/references/philosophy.md
```

**Do NOT create `config.json`** — the skill creates it on first run after exploring the project, proposing a schema, and getting the user's approval at the bootstrap confirmation gate. This ensures the schema is informed by the actual codebase, not a guess.

## Step 3: Adopting the Skill in a Project That Already Has an Inventory

This is an important edge case. The bootstrap path **refuses to overwrite** an existing inventory file — it will error out during pre-flight checks if it detects one.

If the target project already has a feature inventory (even one written by hand), the adoption path is:

1. Do not run bootstrap. The pre-flight check will reject it.
2. Hand-write `config.json` in the installed skill directory. See `references/schema.md` for the shape. You must provide at minimum:
   - `inventory_path` — where the existing inventory file lives
   - `tracked_fields` — which optional fields are populated in your existing entries
   - `vocabularies.status` — the exact `status` values your entries use
   - `vocabularies.personas` — the exact `personas` values your entries use (if the field is tracked)
   - `discovery` — paths where the skill should look for routes, tests, flags, docs
   - `stack.backend` and `stack.frontend` — framework names, used for drift-sync pattern matching
   - `auto_commit` and `commit_message` — preferred commit behavior
3. Run the skill. It will detect `config.json` and run in drift-sync mode immediately, reconciling the existing inventory against the code.

Tell the user about this if they mention having an existing inventory.

## Step 4: First Run (Optional)

Ask the user: "Want me to do a first run now? I'll explore your project, propose a schema tailored to the detected stack, walk the code to draft initial feature entries, and present a single confirmation gate for you to approve or edit."

If they want a first run, execute the installed `SKILL.md`. The bootstrap workflow will handle:

1. Reading `CLAUDE.md`, `README.md`, and manifest files to detect the stack
2. Locating feature surfaces (API routes, frontend routes, tests, flags, docs) using `references/discovery-patterns.md`
3. Inferring a schema and vocabularies tailored to the project
4. Picking an inventory file location based on project conventions
5. Walking the code with the greedy URL-prefix grouping heuristic to propose 10-50 initial entries
6. Presenting one structured confirmation for the user to approve or edit
7. On approval, writing `inventory.yaml` and `config.json`

Bootstrap is **interactive-only**. The skill will fail its pre-flight check if invoked from a scheduled task or cron. Only drift-sync is safe to schedule.

## Step 5: Scheduling (Optional)

Once bootstrap is done, drift-sync can be scheduled to run periodically so the inventory stays in sync as the code evolves.

**Key constraint:** Only schedule drift-sync, never bootstrap. The bootstrap mode requires a human at the confirmation gate and will fail fast if invoked non-interactively.

**Example: Claude scheduled tasks.** Create a scheduled task that invokes the skill:

```markdown
---
name: feature-inventory-sync
description: Periodic drift-sync for the feature inventory
schedule: "0 11 * * 1"  # Every Monday at 11 AM
---

Run the feature-inventory skill in the current project.
The skill will detect that config.json exists and run in drift-sync mode.
```

**Example: cron.** Direct invocation of a headless agent:

```cron
0 11 * * 1 cd /path/to/project && <your-agent-command> "Run the feature-inventory skill"
```

**Example: GitHub Actions.** A workflow that runs on a schedule and opens a PR. Suitable for projects with `auto_commit: false` and `gh` available — see the `auto_commit` behavior documented in `SKILL.md`.

The exact syntax depends on the scheduling framework. The important part: the scheduled task just tells the agent to run the skill, and the skill decides its own mode based on `config.json` presence.
