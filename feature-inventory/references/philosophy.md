# Why a Feature Inventory?

A feature inventory is a single YAML file that catalogs every user-facing capability of a product. This document explains why that catalog is worth maintaining and what design decisions shape it. Read this before running bootstrap — the decisions below are load-bearing for the schema you'll propose.

## The problems it solves

### 1. No single source of truth

In a typical non-trivial codebase, "the set of features" is implicit across at least six different artifacts:

- Frontend routes
- API endpoints
- Database migrations
- Tests
- Documentation
- Feature flags

Nothing ties these together. When a contributor (human or AI) asks "what does this product do?", the only honest answer is "spelunk the code." Every new contributor pays that tax. The inventory pays it once and persists the answer.

### 2. Coverage gaps are invisible

You can list tests. You can list endpoints. What you can't easily answer — without an inventory — is questions like:

- Which features have smoke tests?
- Which features have documentation?
- Which features are still gated by a flag after 6 months?
- Which features touch both frontend and backend?

These are structural questions about coverage, not about individual files. They require a cross-referencing data structure. The inventory is that structure.

### 3. Onboarding is painful

The first thing a new contributor needs is a mental model of the product. The second thing is a sense of what's stable vs. what's changing. A flat list of feature entries with `status: ga|beta|alpha` is the shortest path to both.

This matters for human contributors but it matters more for AI agents. An agent asked to "add client-side filtering to the projects page" has to first find the projects page. An inventory answers that in one file read.

### 4. Auto-generated artifacts need structured input

A changelog between two releases is a diff of the inventory. A coverage matrix is a pivot over the inventory. A capability comparison chart against competitors is a join of your inventory and theirs. None of these are possible without structured input.

The inventory isn't directly useful as auto-generated output — it's a substrate other tools can build on.

### 5. Incident triage requires tracing code paths under pressure

"The `/billing/*` endpoints are returning 500s — what user-facing features are broken?" Without an inventory, answering this means reading router files, tracing imports, and guessing at blast radius while the incident clock ticks. With an inventory, it's a grep for `api_endpoints` containing `/billing/` — you immediately know which features are affected, whether they have fallbacks, and what status they're in.

This is different from coverage gaps (#2). Coverage asks "what's untested?" Blast radius asks "what's on fire and how big is the fire?" Both need the same cross-referencing structure, but the urgency and the audience are different.

### 6. Feature flag rot accumulates silently

Flags get turned on and never cleaned up. A feature shipped behind `new_checkout_flow` six months ago is now just "checkout" — but the flag still exists in the code, the if/else branches still complicate every reader's mental model, and nobody remembers whether it's safe to remove.

An inventory with `feature_flag` fields makes this visible. Any entry where `status: ga` and `feature_flag` is non-null is a candidate for flag cleanup. Without the inventory, discovering these requires cross-referencing the flags table against deployment history — a task nobody does until the flag count is already painful.

### 7. AI agents burn tokens rediscovering what should be a file read

An agent asked to modify a feature needs to answer several questions before writing any code: what endpoints does this feature touch? What's its frontend route? Is it behind a flag? Does it have tests? Are there docs to update?

Without an inventory, the agent explores the codebase to answer each question — reading router files, grepping for test files, checking flag definitions. This exploration consumes tokens, takes time, and risks missing things. With an inventory, the agent reads one file and has a complete map of the feature's surface area. As AI agents become the primary way code gets modified, the inventory becomes less of a nice-to-have documentation artifact and more of a core piece of development infrastructure.

## Design decisions

### Why YAML?

YAML wins on the tradeoffs for this specific shape. The alternatives are all worse in at least one way:

- **JSON**: no comments, no multiline readability, awkward to diff when entries are added/removed
- **TOML**: great for flat configs, awkward for list-of-records
- **Markdown tables**: unparseable reliably, no enforced structure
- **A database**: adds infrastructure for something that should live in git alongside the code

YAML supports comments (useful for section dividers), diffs cleanly in PRs, and every language has a mature parser.

### Why a flat list with comment sections, not nested grouping?

You could write:

```yaml
authentication:
  features:
    - id: login
      ...
project_management:
  features:
    - id: create-project
      ...
```

But this creates problems:

1. **Queries get recursive.** "Does feature `login` exist?" becomes a walk instead of a lookup.
2. **Uniqueness is harder to enforce.** `id` has to be unique globally but live inside a group — duplicates are easy to miss.
3. **Editing gets fiddly.** Moving a feature between groups requires restructuring the YAML.

A flat list with comment section dividers:

```yaml
# ─── Authentication ───
- id: login
  ...

# ─── Project Management ───
- id: create-project
  ...
```

...gets you the grouping affordance without the structural cost. Tools that want to group can do so by parsing `id` prefixes or by looking at the comment dividers.

### Why a single file?

Splitting the inventory into many files (one per category) seems tempting for large projects but loses more than it gains:

- Cross-cutting checks (duplicate IDs, status distribution, "how many beta features do we have?") require walking every file instead of reading one.
- An AI agent reading the inventory has to decide which files to read — a single file fits in a prompt and can be read whole.
- Refactoring a feature across categories becomes an awkward cross-file edit.

For very large products (500+ features), single-file management does start to strain. The skill does not currently handle multi-file inventories — that's a v2 concern.

### Why these specific fields?

The core fields (`id`, `name`, `description`, `status`) are the minimum to answer "what is this feature and what's its state." Everything else is optional because different projects track different things:

- A backend-only service has no `frontend_route`.
- A CLI tool has no `api_endpoints` (in the HTTP sense).
- A solo-dev side project may not have `personas`.
- A pre-PMF startup may not have `docs_page` for anything yet.

Making these optional via `config.tracked_fields` means the inventory shape adapts to the project's actual surface area, not a hypothetical one.

## What counts as a "feature"?

This is the hardest question the bootstrap phase has to answer. There is no objective definition. Common interpretations:

### Interpretation A: One feature per endpoint

Easy to generate automatically. Always produces too many entries. Fails for features that span multiple endpoints (e.g., "login" involves `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`).

### Interpretation B: One feature per URL-prefix group

Groups endpoints by their first URL segment. Produces readable output. Can lump unrelated things together (e.g., `/projects/*` might contain both "project management" and "project file upload" features that really should be separate).

### Interpretation C: One feature per user-facing capability

Matches what users and stakeholders actually talk about. Requires judgment — there's no algorithmic way to detect a "capability." Best for hand-curation, brittle for bootstrap.

**The skill uses Interpretation B as its bootstrap heuristic**, with the user's single confirmation gate as the mechanism for merging/splitting where Interpretation B got it wrong. Over time, as the user refines the inventory during drift-sync runs, the shape drifts toward Interpretation C — curated, not mechanical.

This is deliberate: bootstrap aims for "something reasonable to review," not "the final answer." The user is expected to spend a few minutes during the confirmation gate tightening the proposal.

## When NOT to use an inventory

Not every project needs this. Skip it if:

- **Fewer than ~10 features.** The overhead exceeds the value. A README section is enough.
- **Features are already defined by a formal spec** (OpenAPI, GraphQL schema, protobuf). The spec is already the source of truth; an inventory would duplicate it.
- **Throwaway prototypes.** Stability matters for an inventory to stay useful. If features churn weekly, the inventory will always be stale.
- **Very large products (500+ features)** where a single YAML file becomes unwieldy. Use a database-backed feature registry instead.

## Relationship to other artifacts

The inventory is not a replacement for:

- **API documentation** (OpenAPI, Swagger) — the inventory points to endpoints, it doesn't describe their parameters.
- **User-facing help docs** — the inventory points to docs slugs, it doesn't contain the help content.
- **Roadmap / project tracker** — the inventory describes what exists, not what's planned.
- **Changelog** — though a diff of the inventory across releases gets you most of the way.

Think of the inventory as a **routing table for the product**: given a feature name or ID, it tells you where to look for every artifact connected to that feature. The artifacts themselves live in their own systems.
