# Feature Inventory Schema

The canonical data shape for a feature inventory entry. The skill enforces this shape at bootstrap time and at every drift-sync run.

## Core fields (always required)

Every entry must have these four fields. They are not configurable.

| Field | Type | Description |
|---|---|---|
| `id` | string | Stable kebab-case identifier. Unique across the inventory. Used by tooling to reference a feature. Example: `create-project`, `forgot-password`. |
| `name` | string | Human-readable title. Example: `Create Project`. |
| `description` | string | One-sentence summary of what the feature does. Write it so a new contributor understands the capability without reading any code. |
| `status` | string | Lifecycle state. Must be one of the values in `config.vocabularies.status`. |

## Optional fields

A project opts in to each optional field via `config.tracked_fields`. Entries in the inventory may omit any field that isn't tracked. An entry may also set a tracked field to `null` when the field doesn't apply to that specific feature.

| Field | Type | Purpose |
|---|---|---|
| `personas` | list of strings | Which user types interact with this feature. Values must come from `config.vocabularies.personas`. |
| `frontend_route` | string \| null | Path pattern for the client-side route (e.g., `/projects/:id`). Null if the feature has no frontend surface. |
| `api_endpoints` | list of strings \| null | HTTP routes in `METHOD /path` form. Null or omitted for frontend-only features. |
| `test_file` | string \| null | Relative path to the primary test file. The skill's drift-sync verifies this file exists. |
| `docs_page` | string \| null | Relative path or slug for the feature's documentation. |
| `feature_flag` | string \| null | Name of the flag that gates the feature. Null for features always on. |

Projects are free to add domain-specific optional fields beyond this list by editing `config.tracked_fields` and updating entries to match. The skill will preserve unknown fields during drift-sync but will not generate or validate them.

## Vocabularies

Two fields — `personas` and `status` — use project-defined vocabularies stored in `config.vocabularies`. This is the main hook for keeping the inventory shape out of any particular product's assumptions.

### Status vocabulary

Common choices:
- `[alpha, beta, ga, deprecated]` — classic lifecycle
- `[experimental, stable]` — minimal
- `[internal, beta, ga]` — adds internal tools
- `[ga, beta, internal, sunset]` — tracks end-of-life explicitly

Pick whichever matches how the team already talks about features. The skill will reject any `status` value not in the vocabulary.

### Personas vocabulary

Common choices:
- `[user, admin]` — minimal single-tenant app
- `[owner, member, viewer]` — collaboration tool
- `[student, teacher, admin]` — education product
- `[host, guest, admin]` — two-sided marketplace

Leave empty (`[]`) for products with a single user type. In that case, the `personas` field can be omitted from entries entirely.

## Examples

### Example A — Two-sided product (tutoring marketplace)

```yaml
- id: create-student
  name: Add Student
  description: Tutor adds a new student to their roster
  personas: [tutor]
  frontend_route: /students
  api_endpoints:
    - "POST /api/v1/students/"
  test_file: tests/integration/test_student_onboarding.py
  docs_page: roster/adding-students
  feature_flag: null
  status: ga
```

### Example B — Collaboration tool

```yaml
- id: invite-member
  name: Invite Team Member
  description: Send an email invitation to join the workspace
  personas: [owner, admin]
  frontend_route: /settings/team
  api_endpoints:
    - "POST /api/workspace/invitations"
    - "POST /api/workspace/invitations/:id/resend"
  test_file: tests/team/invitations.test.ts
  docs_page: null
  feature_flag: team_invites_v2
  status: beta
```

### Example C — CLI tool, backend-only

```yaml
- id: export-data
  name: Export Data
  description: Export all workspace data to a JSON file
  frontend_route: null
  api_endpoints:
    - "POST /api/export"
  test_file: tests/export/test_export.py
  docs_page: cli/export
  feature_flag: null
  status: ga
```

Note the omitted `personas` — the project's vocabulary is empty, so no personas field is tracked at all.

### Example D — Frontend-only feature

```yaml
- id: client-side-search
  name: Client-Side Search
  description: Fuzzy search across loaded data without hitting the server
  personas: [user]
  frontend_route: /search
  api_endpoints: null
  test_file: src/search/Search.test.tsx
  docs_page: null
  feature_flag: null
  status: ga
```

## Validation rules

The skill enforces these at every run:

1. `id` must be present, kebab-case, and unique across the inventory.
2. `name` and `description` must be non-empty strings.
3. `status` must be present and must be one of `config.vocabularies.status`.
4. For every field in `config.tracked_fields`, the field must be present on every entry (value can be `null` if not applicable).
5. `personas` values must all be members of `config.vocabularies.personas`.
6. `api_endpoints` entries must match the pattern `METHOD /path`, where `METHOD` is one of `GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS`.
7. Entries with non-null `test_file`, `docs_page`, or `feature_flag` must point to targets the skill can verify during drift-sync (file exists, flag defined).

Failures are reported as structured warnings during drift-sync. The skill does not auto-fix these — it surfaces them so a human can decide.

## File format conventions

The inventory is a YAML document containing a single top-level list. Comments are encouraged as section dividers:

```yaml
# <Product Name> — Feature Inventory
# Single source of truth for every user-facing capability.
# Maintained by the feature-inventory skill.

# ─── Authentication ─────────────────────────────────────────────
- id: login
  name: Login
  ...

# ─── Core Domain ───────────────────────────────────────────────
- id: create-record
  ...
```

**Why YAML?** Comments, readability, diff-friendliness. JSON has no comments. TOML doesn't handle list-of-records gracefully. YAML is the least-bad choice for this shape.

**Why a flat list?** Nested grouping complicates every query ("does this feature exist?" becomes a recursive walk). Comment dividers give section structure without sacrificing grep-ability.

**Why a single file?** Cross-cutting checks (duplicate IDs, status distribution, coverage ratios) are trivial when the whole inventory is in one document. For projects with 500+ features this may become unwieldy — the skill does not yet handle multi-file inventories.
