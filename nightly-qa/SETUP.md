# Nightly QA — Setup Guide

You are setting up an automated nightly QA skill for the user's project. This skill runs E2E browser tests against local dev servers using Chrome DevTools MCP, writes reports, fixes bugs, and improves itself over time.

Follow these steps in order. Ask the user for input only when you can't infer the answer from the codebase.

## Step 1: Prerequisites Check

Before starting, verify these are available:

1. **Chrome DevTools MCP** — check if the project or user has a Chrome DevTools MCP server configured (look for `chrome-devtools` in `.mcp.json`, MCP config files, or equivalent)
   - If not found: tell the user they need to set up the Chrome DevTools MCP server and stop here until resolved
2. **Git repo** — verify the project directory is a git repository
3. **gh CLI** — verify `gh` is installed (`gh --version`). Needed for creating PRs when bugs are found.
   - If not found: tell the user to install it (https://cli.github.com) or note that the PR creation step will need to be done manually
4. **Database** — check if the project uses a database (look for ORM configs, migration files, connection strings). Note the DB type (PostgreSQL, MySQL, SQLite, etc.)

## Step 2: Discover the Project

Read the codebase to understand the project structure. Gather:

### Tech stack
- Read `package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`, or equivalent
- Identify: frontend framework (React, Vue, Svelte, etc.), backend framework (FastAPI, Express, Django, Rails, etc.), language(s)
- Find the frontend and backend entry points

### Dev server configuration
- Find how dev servers are started (npm scripts, Makefile, docker-compose, etc.)
- Identify the ports (check scripts, config files, or hardcoded values)
- Check if there's an existing tmux/screen setup

### Auth mechanism
- Search for auth middleware, session management, login routes
- Identify: cookie-based sessions, JWT, OAuth, API keys, or no auth
- Find the cookie/header names used for authentication
- Look for any dev auth helpers or test fixtures that create sessions

### Routes and pages
- Find the router configuration (React Router, file-based routing, backend routes, etc.)
- List all user-facing routes/pages
- Identify which routes require authentication

### Database schema
- Read migration files, ORM models, or schema definitions
- Identify the main entities and their relationships
- Map out foreign key dependencies (needed for cleanup queries)

### Existing test infrastructure
- Check for existing E2E tests, test utilities, seed data scripts
- Look for test database configuration
- Find any existing QA or testing documentation

## Step 3: Ask the User

Present what you discovered and ask for confirmation/additions:

```
Here's what I found about your project:

- **Tech stack**: [frontend] + [backend]
- **Frontend port**: [port]
- **Backend port**: [port]
- **Base URL for testing**: [url]
- **Auth mechanism**: [type] — cookie name: [name]
- **Routes found**: [list key routes]
- **Database**: [type] — [entity count] main entities

I need your input on a few things:

1. **Which flows are critical to test nightly?** I found these routes: [list].
   Which are the most important user journeys? (e.g., "signup → onboarding → create first project → invite team member")

2. **How do I create a test auth session?** [If no dev auth helper found: "I didn't find a dev auth helper. How do authenticated users get sessions — is there a script, or should I go through the login UI?"]

3. **Where should QA reports be saved?** Suggestion: `docs/testing/nightly-qa-reports/`

4. **What time should this run?** Default: 10 PM daily

5. **Any pages or flows I should NOT interact with?** (e.g., billing, external integrations, email-sending endpoints)

6. **Is there seed/demo data I should use for read-only verification?** (e.g., demo users, sample records)
```

Wait for the user's answers before proceeding.

## Step 4: Generate the SKILL.md

Use the template in `SKILL.md` from this directory as your base. Customize it:

### 4a. Fill in the Configuration section
Replace the example values with the actual values discovered + confirmed by the user.

### 4b. Write the test flows
This is the most important part. Generate real, project-specific test flows based on:
- The critical user journeys the user identified
- The routes and pages you discovered
- The database schema (for knowing what to create/verify/cleanup)

**Structure the flows into phases:**
- **Phase 1: Authentication** — signup, login, password reset, session handling
- **Phase 2: Core workflows** — the main CRUD operations and user journeys (authenticated)
- **Phase 3: Secondary workflows** — less critical but still important features
- **Phase 4: Error resilience** — expired sessions, 404s, invalid IDs, edge cases
- **Phase 5: Page smoke tests** — lightweight render checks for all remaining pages

**For each test flow, be specific:**
- Use actual route paths from the codebase
- Reference actual form field names/IDs/placeholders from the components
- Use actual button text from the UI code
- Use actual API endpoint paths for network verification
- Prefix all created test data with the `TEST_DATA_PREFIX`
- Use `TEST_EMAIL_DOMAIN` for any email addresses

**Read the actual component files** to get the correct:
- Input field selectors (name, id, placeholder, aria-label)
- Button text
- Validation error messages
- Success/error toast messages
- Modal/dialog content

### 4c. Write the cleanup section
Based on the database schema, generate FK-ordered DELETE statements that clean up all test data created during the run. Be thorough — missed cleanup causes stale data that breaks future runs.

### 4d. Write the auth setup section
Based on the auth mechanism, write the specific commands to create test sessions.

## Step 5: Install the Skill

Save the generated SKILL.md to wherever the user's agent reads skill files from. Common locations:
- `~/.claude/scheduled-tasks/nightly-qa/SKILL.md` (Claude Code)
- The project's `.agent/skills/` directory
- Wherever the user specifies

Also copy the `references/` directory alongside the SKILL.md — it contains gotchas that the skill references during runs.

Create the report directory in the project:
```bash
mkdir -p <PROJECT_ROOT>/<REPORT_DIR>
```

## Step 6: Schedule It

Ask the user what time they'd like it to run, then help them set up the schedule using their agent's scheduling mechanism (cron, built-in scheduler, or equivalent).

## Step 7: Verify

Do a dry run of the pre-flight checks to make sure everything works:

1. Check that dev servers are reachable on the configured ports
2. Verify Chrome DevTools MCP tools are available (try `list_pages`)
3. Create and immediately delete a test record to verify DB access and cleanup
4. Create the first (empty) report file to verify the report directory is writable

Report the results to the user. If everything passes, the skill is ready for its first real run.

## Step 8: First Run (Optional)

Ask the user: "Want me to do a first run now to validate the full test suite, or wait for the scheduled time?"

If they want a first run, execute the SKILL.md you just installed.
