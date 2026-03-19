---
name: nightly-qa
description: Automated E2E browser testing via Chrome DevTools MCP. Runs nightly against local dev servers — checks console errors, network requests, DOM state. Writes checkpoint reports, fixes bugs, opens PRs, and improves itself after each run. Use when setting up recurring QA testing for a web application.
compatibility: Requires Chrome DevTools MCP server, a browser, local dev servers (frontend + backend), git, and gh CLI (for PR creation). Database access needed for test data cleanup.
metadata:
  author: yashasvigirdhar
  version: "1.0"
---

# Nightly QA — E2E Browser Tests

You run automated QA tests against the local dev environment using Chrome DevTools MCP tools. Tests cover the flows defined in the **Test Flows** section below.

**Key principle — MANDATORY checkpointing**: After EVERY SINGLE test step, you MUST immediately append the result to the report file. Do NOT batch results. Do NOT hold results in memory to write later. The report file is your checkpoint — if context compresses mid-run, you can read it back to see what you've already done. Treat each write as a save point. If you lose context, read the report file to recover your position and continue from the next untested step.

## Configuration

Before using this skill, set these values for your project:

| Variable | Description | Example |
|---|---|---|
| `PROJECT_ROOT` | Absolute path to your project | `/home/you/projects/myapp` |
| `FRONTEND_PORT` | Port your frontend dev server runs on | `3000` |
| `BACKEND_PORT` | Port your backend dev server runs on | `8000` |
| `BASE_URL` | URL used for browser testing | `http://localhost:3000` |
| `REPORT_DIR` | Where QA reports are saved (relative to project root) | `docs/testing/nightly-qa-reports` |
| `TMUX_SESSION` | Name of the tmux session running dev servers | `devservers` |
| `AUTH_SETUP_CMD` | Command to create test auth sessions (if applicable) | `python -m scripts.create_session --user-type admin` |
| `TEST_DATA_PREFIX` | Prefix for QA-created test data (for cleanup) | `[QA-NIGHTLY]` |
| `TEST_EMAIL_DOMAIN` | Non-routable email domain for test accounts | `test.local` |

## Before You Start

1. Read the **Accumulated Learnings** section at the bottom of this file. Apply any lessons from previous runs to avoid repeating mistakes.
2. Read the most recent report in `REPORT_DIR` (if any). Extract two things:
   - Context on past results and recurring issues
   - The **Current commit** hash from that report — this becomes your `Last tested commit` for today's Change Analysis
3. Read any project-specific verification protocol docs (if referenced in your test flows).

## Pre-flight Checks

1. **Ensure dev servers are running** (they run in tmux session `TMUX_SESSION`):
   - Check if tmux session exists: `tmux has-session -t TMUX_SESSION 2>/dev/null`
   - Check ports: `lsof -i :FRONTEND_PORT` (frontend), `lsof -i :BACKEND_PORT` (backend)
   - **If tmux session doesn't exist**, create it and start both servers:
     ```bash
     tmux new-session -d -s TMUX_SESSION -c PROJECT_ROOT
     # Window 0: backend
     tmux send-keys -t TMUX_SESSION:0 '<your backend start command>' Enter
     # Window 1: frontend
     tmux new-window -t TMUX_SESSION -c PROJECT_ROOT/frontend
     tmux send-keys -t TMUX_SESSION:1 '<your frontend start command>' Enter
     ```
   - **If tmux session exists but a server is down**, restart just that server in its existing tmux window.
   - **Wait for servers to be ready**: poll `lsof -i :FRONTEND_PORT` and `lsof -i :BACKEND_PORT` every 3 seconds, up to 30 seconds. If still not up after 30s, create a report with status "ABORTED: servers failed to start", commit it, and stop.
   - Note in the report whether servers were already running or had to be started.
2. Check for stale test data from a previous failed run:
   - Query your database for any records matching `TEST_DATA_PREFIX`
   - If any found, clean them up before proceeding (see Cleanup section).

## Change Analysis

After pre-flight checks, before creating the report, analyze what changed since the last run. This gives you context to adapt your testing — paying extra attention to changed areas and discovering new functionality to verify.

**Step 1: Find the last tested commit**
- Read the most recent report in `REPORT_DIR`. Look for the `Last tested commit:` line in the header.
- If found: use that as the base. `git log <last-commit>..HEAD --no-merges --oneline`
- If not found (first run or missing): fall back to `git log --since="24 hours ago" --no-merges --oneline`

**Step 2: Get an overview of what changed**
```bash
git diff <base>..HEAD --stat -- frontend/ backend/
```
This shows which files changed and how many lines. Focus on frontend files and API endpoints — skip docs, scripts, config.

**Step 3: Read the actual diffs (selectively)**
For each frontend file with >10 lines changed, read the full diff:
```bash
git diff <base>..HEAD -- <file>
```
**Context window budget**: Cap total diff reading at ~500 lines. Prioritize in this order:
1. Page-level changes (most impactful)
2. Component changes
3. API endpoint changes (informational — note what changed but don't read full diff unless it's a new file)

If a busy period produced >500 lines of frontend diff, read the page files fully and summarize the component changes from `--stat` output.

**Step 4: Understand and plan**
After reading the diffs, build your understanding:
- What new UI elements were added? (buttons, dialogs, sections, fields)
- What existing behavior was modified? (validation changes, layout changes, renamed elements)
- Were any new routes added?
- Were any API endpoints added or modified?

**Step 5: Write the Change Summary to the report immediately**
This is a checkpoint — write it to the report file NOW before running any tests. Format:
```markdown
## Recent Changes (since last run)

**Commits analyzed**: N commits from `<base-hash>` to `<head-hash>`

### Changes affecting tests:
- `<hash>` — <commit message>
  - `<file>`: <what changed — e.g., "Added bulk delete button, checkbox selection state">
  - **Affects test(s)**: #N (<test name>)

### New functionality to test (ad-hoc):
- <description> — will verify during test #N
- <description> — new route `/foo`, will smoke test

### No-impact changes (skipped):
- `<hash>` — <commit message> (docs/config/backend-only)
```

If no commits since last run: write "No changes since last run. Running standard test suite." and skip to test execution.

**Step 6: Adaptive testing during the run**
When you reach a test that exercises changed code, perform the standard steps PLUS ad-hoc verification of the new functionality you identified in Step 4. Note these ad-hoc checks in the report as `[AD-HOC]` so they're distinguishable from standard steps.

## Report Setup

**IMMEDIATELY** create the report file before running any tests.

File: `REPORT_DIR/YYYY-MM-DD.md` (use today's date)

Initial content:
```
# Nightly QA Report — YYYY-MM-DD

- **Started**: HH:MM
- **Environment**: local dev (BASE_URL → localhost:BACKEND_PORT)
- **Last tested commit**: <commit hash from previous report, or "first run">
- **Current commit**: <current HEAD hash>
- **Previous run**: [link to previous report or "first run"]
- **Learnings applied**: [list any learnings from Accumulated Learnings section that were relevant]

## Recent Changes (since last run)

[Write the Change Analysis output here — see Change Analysis section above]

## Test Results
```

After EACH test step, immediately append the result to this file:
```
### N. Flow Name
- **Status**: PASS / FAIL
- **Details**: what was verified, what interactions were performed
- **Console errors**: none / [list errors]
- **Network failures**: none / [list failed requests with status codes]
- **Screenshot**: [path if taken, screenshots taken on FAIL only]
```

## Auth Setup

If your app requires authentication for testing, create sessions before running tests.

```bash
AUTH_SETUP_CMD
```

Store session IDs and tokens. Inject them as browser cookies before navigating to authenticated pages:
```javascript
document.cookie = "session=<session_id>; path=/";
```

When switching between user roles (e.g., admin → regular user), always clear old session cookies before injecting new ones to avoid conflicts.

## Test Flows

**Base URL**: `BASE_URL`

**Verification protocol for EVERY step:**
1. Perform the action (navigate, click, fill, submit)
2. Check console errors: `list_console_messages(types: ["error"])`
3. Verify DOM state: `take_snapshot()` and/or `evaluate_script()`
4. Check network: `list_network_requests(resourceTypes: ["fetch", "xhr"])` — verify no 4xx/5xx on expected calls
5. On FAIL: `take_screenshot()` and save to `REPORT_DIR/screenshots/YYYY-MM-DD-step-N.png`
6. **Append result to report immediately** — this is non-negotiable

**Phase checkpoint protocol**: At the END of each phase (after all tests in that phase), write a phase summary line to the report:
```
### Phase N Complete — M/M passed
```
Then **read the report file back** to verify your checkpoint is persisted. This ensures that if context compresses during the next phase, you can recover by reading the report. Do NOT skip the read-back — it costs almost nothing and prevents losing an entire phase of results.

**Context recovery protocol**: If at any point you feel uncertain about what you've already tested (e.g., after context compression), STOP and read the report file. It is the source of truth. Resume from the next untested step.

**Be thorough**: Each test below lists specific interactions and verifications. You MUST perform ALL of them, not just the first one or two. Skipping steps defeats the purpose of nightly QA. If a step fails, note it in the report and continue to the next step — do not skip remaining verifications just because one failed. The goal is to find ALL bugs per run, not stop at the first one.

---

<!-- ============================================================
     YOUR TEST FLOWS GO HERE

     Organize tests into phases. Each phase groups related flows.
     Each flow has a number, name, and specific verification steps.

     Below is an example structure — replace with your actual tests.
     ============================================================ -->

### Phase 1: Authentication

**1. Signup Page Rendering**
- Navigate to `/signup`
- Verify: form fields are present (name, email, password)
- Verify: social login buttons are visible (if applicable)
- Verify: "Sign in" link exists
- Verify: no console errors, no 5xx network responses

**2. Signup — Validation**
- Leave all fields empty, click submit → verify validation errors appear
- Fill invalid email → verify email validation
- Fill short password → verify password validation

**3. Signup — Happy Path**
- Fill form: email=`qa-nightly-YYYYMMDD@TEST_EMAIL_DOMAIN`, name=`TEST_DATA_PREFIX Test User`, password=`QaNightly2026!`
- Submit the form
- Verify: no error, redirects away from `/signup`
- Verify: no console errors, no 5xx network responses

**4. Cleanup Signup Test**
- Delete the test user from the database immediately

---

### Phase 2: Core Workflows (Authenticated)

Inject auth session cookie, navigate to the app.

**5. Dashboard**
- Verify: page renders without errors
- Verify: key sections are visible (list yours)

**6. CRUD Flow — Create**
- Navigate to the relevant page
- Click create button → verify dialog/page opens
- Fill required fields with `TEST_DATA_PREFIX` prefixed data
- Submit → verify success

**7. CRUD Flow — Read**
- Verify the created item appears in the list
- Open detail view → verify data matches

**8. CRUD Flow — Update**
- Edit the item → change a field → save
- Verify: updated value is reflected

**9. CRUD Flow — Delete**
- Delete the item → confirm deletion
- Verify: item is removed from list

---

### Phase 3: Error Resilience

**10. Expired Session Redirect**
- Clear all cookies
- Navigate to a protected route
- Verify: redirects to login (not a white screen or crash)

**11. 404 Page**
- Navigate to `/this-page-does-not-exist-qa-test`
- Verify: proper Not Found UI renders
- Verify: no unhandled JS exceptions

**12. Invalid Resource ID**
- Navigate to a resource detail page with a fake UUID
- Verify: error state renders gracefully

---

### Phase 4: Page Smoke Tests

Lightweight render checks for pages not covered above.

**13. Page A**
- Navigate to `/page-a`
- Verify: page renders, no console errors

**14. Page B**
- Navigate to `/page-b`
- Verify: page renders, no console errors

<!-- END OF TEST FLOWS -->

---

## Cleanup

**CRITICAL**: Always run cleanup, even if tests failed mid-way.

Delete all test data matching `TEST_DATA_PREFIX` and `TEST_EMAIL_DOMAIN` from your database.

Example (adapt to your schema):
```sql
-- Delete test users and their dependent data (respect FK order)
DELETE FROM dependent_table WHERE user_id IN (SELECT id FROM users WHERE email LIKE 'qa-nightly%@TEST_EMAIL_DOMAIN');
DELETE FROM users WHERE email LIKE 'qa-nightly%@TEST_EMAIL_DOMAIN';
```

Verify cleanup:
```sql
SELECT COUNT(*) AS remaining FROM users WHERE email LIKE 'qa-nightly%@TEST_EMAIL_DOMAIN';
-- Must be 0
```

Append cleanup result to report.

## Report Finalization

Append the summary section to the report:

```
## Summary

- **Completed**: HH:MM
- **Duration**: X minutes
- **Total flows**: N
- **Passed**: N
- **Failed**: N
- **Bugs found**: N
- **Fixes applied**: N
- **Cleanup**: PASS / FAIL

## Learnings

- [What went well in this run]
- [What was tricky — specific selectors that were hard to find, timing issues, unexpected UI states]
- [Differences from previous runs — what changed in the app since last time]
- [Suggestions for improving this test procedure]
- [Any flaky behavior observed — intermittent failures, race conditions]
```

## Bug Fix & PR Flow

If any test revealed a **code bug** (not an environment issue, not a test procedure issue, not a transient/timing problem):

1. Analyze the root cause from console errors, network responses, and DOM state
2. Read the relevant source code to understand and fix the bug
3. Make the fix
4. Re-run ONLY the failing test step(s) to verify the fix works
5. Update the report with fix details under a `## Bugs Fixed` section
6. Create a branch and PR:
   ```bash
   git checkout -b qa/nightly-fix-YYYY-MM-DD
   git add <fixed-files> REPORT_DIR/
   git commit -m "Fix <short description> (found by nightly QA)"
   gh pr create --title "Fix: <short description> (nightly QA YYYY-MM-DD)" --body "$(cat <<'EOF'
   ## Summary
   - Found by nightly QA run on YYYY-MM-DD
   - <what was broken and how it was fixed>

   ## Test plan
   - [ ] Verified fix passes in nightly QA re-run
   - [ ] No regressions in other test flows
   EOF
   )"
   git checkout main
   ```

If **NO bugs found**, commit just the report:
```bash
git add REPORT_DIR/
git commit -m "Nightly QA report YYYY-MM-DD — all tests passed"
```

## Self-Improvement

After finalizing the report, review the "Learnings" section and update THIS SKILL.md file:

1. Read today's learnings
2. If any learning is **generalizable** and would help future runs, add it to the **Accumulated Learnings** section below
3. If a test step needs adjustment (wrong selector, wrong URL, different UI flow, better verification approach), update the relevant step above
4. If a flow has changed significantly in the app, update the test instructions to match
5. Remove any accumulated learnings that are no longer relevant (e.g., a bug was fixed, a UI was redesigned)

**Promote ad-hoc steps**: Review any `[AD-HOC]` checks you added during this run based on the Change Analysis. If the check was useful and the functionality is permanent (not a WIP), add it as a permanent step in the relevant test. This is how the test suite grows to match the app — new code → ad-hoc check → permanent test step.

**Prune stale tests**: If a test step references UI that no longer exists (you know this from the Change Analysis diffs or from test failures where the element was removed), remove or update that step. Don't leave dead test steps in the SKILL.md.

The goal: each run makes the next run smoother, more reliable, AND more comprehensive.

---

## Accumulated Learnings

_This section is maintained by the agent after each run. It captures reusable knowledge that improves future test execution. Each entry should include the date it was learned and why it matters._

Refer to [references/chrome-devtools-gotchas.md](references/chrome-devtools-gotchas.md) for common Chrome DevTools MCP pitfalls (cookie injection, React controlled inputs, session management, context window tips).
