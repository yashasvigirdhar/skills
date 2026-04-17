---
name: feature-inventory
description: "Use when the user wants to create or maintain a YAML feature inventory for a product — a single source of truth cataloging every user-facing capability. Handles both initial bootstrap (explores the codebase to propose a schema and initial entries) and ongoing drift-sync (reconciles the inventory against the current code)."
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
compatibility: Works with any skills-compatible agent (Claude Code, Cursor, Copilot, etc.). No external accounts or API keys needed. Git optional — only required if you want auto-commit or PR creation during drift-sync.
license: MIT
metadata:
  author: yashasvigirdhar
  version: "1.0"
---

# Feature Inventory — Create & Maintain

You catalog every user-facing capability of a product in a single YAML file, then keep it in sync with the actual codebase over time.

The inventory is the single source of truth that ties frontend routes, API endpoints, tests, docs, and feature flags together into one structured list. It is the foundation for coverage matrices, auto-generated docs, onboarding, and agent-readable product context.

## Before You Start

1. Read `references/philosophy.md` — understand what counts as a "feature" and why the inventory matters. This is load-bearing for bootstrap decisions.
2. Read `references/schema.md` — the canonical data shape. You must produce entries that conform to it.
3. Read `references/discovery-patterns.md` when you need to locate routes, tests, or flags in an unfamiliar stack.
4. Read `runs.log` in this skill directory if it exists — prior runs may contain stack-specific learnings that save you work.
5. Read the `## Accumulated Learnings` section at the bottom of this file.

## Deciding Which Mode to Run

Check whether `config.json` exists in this skill directory (the same directory as this `SKILL.md`).

| State | Mode |
|---|---|
| `config.json` missing | **Bootstrap mode** — create the inventory from scratch |
| `config.json` present | **Drift-sync mode** — reconcile existing inventory against code |

These are two distinct workflows. Do not try to do both in a single run.

---

## Mode 1: Bootstrap

Bootstrap creates a new `config.json` and a new `inventory.yaml` from scratch. It requires a human at the confirmation gate.

### Pre-flight checks

**Check 1 — Interactive context.** If you are running inside a scheduled task, cron job, or any non-interactive context, stop and report:

> Bootstrap requires a user at the confirmation gate. Run this skill interactively the first time to create `config.json` and `inventory.yaml`, then schedule drift-sync for subsequent runs.

**Check 2 — Existing inventory.** Use `Glob` to search for any existing inventory file in the project:

```
**/inventory.yaml
**/feature-inventory.yaml
docs/feature-inventory/*.yaml
```

If any file looks like an existing feature inventory (contains a list of entries with `id` and `name` fields), stop and report:

> An inventory file already exists at `<path>`. This skill's bootstrap mode assumes a clean slate and will not overwrite. Two options:
>
> 1. Remove the existing file and rerun bootstrap.
> 2. Keep the existing file, hand-write `config.json` in this skill directory to match your current schema (see `references/schema.md`), then rerun — the skill will detect `config.json` and switch to drift-sync mode.

Do not proceed with bootstrap if either check fails.

### Step 1: Explore the project

Read in this order:
1. `CLAUDE.md` and `AGENTS.md` in the project root (if present)
2. `README.md` in the project root
3. Manifest files that identify the stack: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, `composer.json`
4. Top-level directory listing

From these, form a working hypothesis: what backend framework, what frontend framework, what test layout, what docs framework. Record these.

### Step 2: Locate feature surfaces

For each of the following, locate the relevant file(s) or directory. Use `Glob` and `Grep` to find candidates, then verify by reading. Consult `references/discovery-patterns.md` for stack-specific hints.

- **API routes / endpoints** — where are HTTP routes registered?
- **Frontend routes** — where are client-side routes defined?
- **Tests** — where do tests live? Unit, integration, smoke, E2E?
- **Feature flags** — is there a flags module, a flags table, or a third-party service?
- **Documentation source** — where are docs markdown files?

You will not always find all of these. A backend-only service has no frontend routes. A CLI tool has no HTTP routes. Record what you found and skip what you didn't.

### Step 3: Infer the schema

Based on what you found, decide which optional fields to track in the inventory:

- Found API endpoints → track `api_endpoints`
- Found frontend routes → track `frontend_route`
- Found a test layout → track `test_file` (or rename to match local terminology, e.g., `smoke_test`, `integration_test`)
- Found flags → track `feature_flag`
- Found a docs source → track `docs_page`

Decide on vocabularies. These go into `config.json` so the skill can validate later entries.

- **`personas`** — scan auth middleware, type definitions, role constants, seed data. Look for enum-like values such as `ADMIN`, `USER`, `TUTOR`, `STUDENT`. If you find explicit roles, use them. If you find nothing, propose `[user, admin]` and **explicitly flag this as an assumption** in the confirmation summary so the user can correct it.
- **`status`** — default to `[alpha, beta, ga, deprecated]` unless the project's README, CHANGELOG, or docs use different lifecycle terms. Record the chosen values.

### Step 4: Pick an inventory file location

Choose the location yourself based on the project's conventions. Rules of thumb:

- If `docs/` exists at the repo root → `docs/feature-inventory/inventory.yaml`
- Else if `docs/` is absent but there's a `documentation/` or similar → use that parent
- Else → `feature-inventory.yaml` at the repo root

Do not ask the user about this separately. Your choice goes into the single bootstrap confirmation, where the user can push back.

### Step 5: Walk the code, propose initial entries

Use the greedy grouping heuristic to keep the initial list reviewable:

1. Scan API endpoint definitions. Group endpoints by the first URL segment (e.g., all `/users/*` endpoints become one candidate feature).
2. Scan frontend routes. For each route, check whether an endpoint group covers the same conceptual area (by name or URL prefix). If yes, attach the route to that group as its `frontend_route`. If no, the route becomes a **frontend-only feature**.
3. Any endpoint group with no matching frontend route is a **backend-only feature**.
4. For each proposed entry, draft:
   - `id`: kebab-case, derived from the URL prefix or route name
   - `name`: Human-readable title
   - `description`: Inferred from docstrings, route handler comments, or the URL/route itself. One short sentence.
   - `status`: Your best guess. Default to `ga` if the feature appears to be in production; `beta` or `alpha` if you see clear signals (feature flag, `WIP`, `experimental` marker).
   - All optional fields populated from what you found.

Propose between 10 and 50 entries as a starting skeleton. Do not attempt to be exhaustive — the user can add more later. Over-proposing exhausts the confirmation budget.

### Step 6: Present ONE confirmation

Output a single structured summary as your response. Do not ask any field-by-field questions. The user will either approve everything or push back with specific edits.

Structure:

```
## Feature Inventory Bootstrap — Review

### Proposed inventory location
<path>

### Detected stack
- Backend: <framework or "none">
- Frontend: <framework or "none">
- Tests: <location or "none">
- Flags: <location or "none">
- Docs: <location or "none">

### Proposed schema (tracked fields)
- Core: id, name, description, status
- Optional: <list only the fields you're including>

### Proposed vocabularies
- personas: [<values>]   <flag as "inferred" or "assumed — no role terminology found" if guessed>
- status: [<values>]

### Proposed discovery paths (used for future drift-sync)
- api_endpoints: <path>
- frontend_routes: <path>
- tests: <path>
- flags: <path>
- docs: <path>

### Proposed entries (<N> features)

1. <id> — <one-line summary>
2. <id> — <one-line summary>
...

### Explicit assumptions
- <List anything you had to guess: inferred vocabularies, ambiguous file locations, uncertain descriptions>

Ready to write? Push back on anything wrong and I'll re-present. Otherwise say "go" and I'll create `inventory.yaml` and `config.json`.
```

After printing this, **stop**. Do not call any more tools. Wait for the user's next message.

### Step 7: Handle feedback and re-present

If the user points out errors, make the specific edits and re-present the updated plan. Repeat until they approve.

Accept edits like:
- "Change personas to `[owner, member, viewer]`"
- "Merge features 3 and 4 into one entry called 'account management'"
- "Drop features 7, 12, and 15"
- "Use `docs/inventory.yaml` instead"

Do not take fresh actions (scanning new files, proposing new features) unless the user explicitly asks.

### Step 8: On approval, write files

1. Create any parent directories needed for the inventory file.
2. Write `inventory.yaml` at the chosen path. Include a header comment linking back to this skill's `references/schema.md` and `references/philosophy.md` so future readers know where the format comes from. Group entries with comment section dividers where natural (auth, users, billing, etc.).
3. Write `config.json` in this skill directory:

   ```json
   {
     "inventory_path": "<path to inventory.yaml>",
     "tracked_fields": ["<list from Step 3>"],
     "vocabularies": {
       "personas": ["<values>"],
       "status": ["<values>"]
     },
     "discovery": {
       "api_endpoints": "<path>",
       "frontend_routes": "<path>",
       "tests": "<path>",
       "flags": "<path>",
       "docs": "<path>"
     },
     "stack": {
       "backend": "<framework name>",
       "frontend": "<framework name>"
     },
     "auto_commit": true,
     "commit_message": "chore: sync feature inventory with codebase"
   }
   ```

   Omit any `discovery` keys for surfaces that don't apply (e.g., no `frontend_routes` for a backend-only project).

4. **Do not commit.** Bootstrap leaves the files staged for the user to review in their own git workflow.

5. Print a short completion summary pointing at both files and telling the user that subsequent runs will switch to drift-sync mode automatically.

---

## Mode 2: Drift-sync

Drift-sync reconciles an existing inventory against the current state of the code. It can run interactively or non-interactively (in a scheduled context).

### Step 1: Load state

Read `config.json` in this skill directory. Then read the inventory file at `config.inventory_path`. If either file is missing or malformed, fail with a clear error.

### Step 2: Detect drift

For each tracked field, compare inventory against code using the paths in `config.discovery`. Apply your knowledge of the stack (recorded in `config.stack`) — see `references/discovery-patterns.md` for patterns specific to common frameworks.

**API endpoints drift:**
- Build a set of routes in code by scanning the discovery location
- Build a set of routes referenced in the inventory (flatten all `api_endpoints` lists)
- Routes in code but not inventory → drift: new endpoint, needs to be added to an existing feature or a new entry
- Routes in inventory but not code → drift: removed endpoint, entry may need cleanup or deletion

**Frontend routes drift:** same approach — compare routes in the router file against `frontend_route` values.

**Test file drift:** for each entry with a non-null `test_file`, verify the file exists at the expected path. Flag missing files.

**Docs page drift:** for each entry with a non-null `docs_page`, verify the file exists.

**Feature flag drift:** for each entry with a non-null `feature_flag`, verify the flag still exists in the flags source.

**Structural checks:**
- Duplicate `id` values
- `status` values not in the configured vocabulary
- `personas` values not in the configured vocabulary

### Step 3: Propose the diff

Present the changes before applying them. Structure:

```
## Feature Inventory Drift Detected

### Added (<N>)
  + <id> — <what changed and why>

### Removed (<M>)
  - <id or endpoint> — <what's gone and why>

### Changed (<P>)
  ~ <id> — <field change>

### Warnings (<Q>)
  ! <issue description>
```

If no drift: print `Inventory is in sync. No changes.` and exit.

### Step 4: Apply changes

Behavior depends on `config.auto_commit` and whether the `gh` CLI is available.

**`auto_commit: true` (default):**
1. Write changes to the inventory file
2. `git add <inventory_path>`
3. `git commit -m "<config.commit_message>"`
4. Print commit hash
5. Do not push (the user or scheduler decides when to push)

**`auto_commit: false` AND `gh` CLI is not available:**
1. Write changes to the inventory file
2. Leave them in the working tree uncommitted
3. Print: "Changes written to `<path>`. Review and commit when ready."

**`auto_commit: false` AND `gh` CLI is available:**
1. Save the current branch name
2. Create a new branch: `feature-inventory-sync-<YYYY-MM-DD>`
3. Write changes to the inventory file
4. `git add <inventory_path>`
5. `git commit -m "<config.commit_message>"`
6. `git push -u origin <branch>`
7. Open a PR with `gh pr create`:
   - Title: `<config.commit_message>`
   - Body: structured list of the diff (added/removed/changed/warnings sections from Step 3)
8. Return to the original branch
9. Print the PR URL

To check whether `gh` is available, run `command -v gh` via the Bash tool. If the exit code is non-zero, treat `gh` as unavailable.

### Step 5: Report

Print a one-line summary of what was detected and what action was taken.

---

## Known Limitations

- **Greedy grouping during bootstrap can over-lump.** The URL-prefix heuristic sometimes puts unrelated endpoints in the same feature. The confirmation gate is the correction mechanism — users are expected to split/merge during review.
- **Static route parsing has blind spots.** Dynamically registered routes (loop-based, meta-programming) may not be detected. For frameworks that support it, prefer runtime introspection during drift-sync.
- **Third-party feature flag services are opaque.** If flags live in PostHog, LaunchDarkly, etc., the skill can only track them by name — it cannot verify they still exist in the service. Configure `config.stack.flag_service` to skip the local file check.
- **No multi-file inventory support.** For projects with 500+ features, the single-file approach will become strained. Not currently supported.
- **Bootstrap refuses to overwrite existing inventory files.** This is intentional — it prevents silent clobbering of hand-curated data. Use the "hand-write config.json" escape hatch described in `SETUP.md` if you want to adopt the skill in a project that already has an inventory.

## After the Run

Regardless of mode, do two things:

### 1. Append to `runs.log`

Create the file if it doesn't exist. Append an entry:

```
## YYYY-MM-DD HH:MM — <mode>
- Mode: <bootstrap|drift-sync>
- Outcome: <short summary>
- Files touched: <list>
- Drift detected: <counts for drift-sync; "N/A" for bootstrap>
- Action taken: <committed / PR opened / left in working tree / no-op>
- Learnings: <anything non-obvious that would help a future run>
```

### 2. Update Accumulated Learnings if warranted

If you learned something **generic** about the workflow itself (not specific to this project), append an entry to the section below. Focus on:
- Discovery patterns that worked for an unfamiliar stack
- Grouping heuristics that produced better bootstrap proposals
- Edge cases in drift detection

Do **not** record project-specific paths, vocabularies, or examples in Accumulated Learnings. Those belong in the user's `config.json` or `runs.log`, not in the skill's shared memory.

## Accumulated Learnings

_This section is maintained by the agent after each run. Each entry should include the date, what was learned, and why it matters. Keep entries generic — project-specific details belong in runs.log._

(empty — no runs yet)
