# Discovery Patterns

Reference for locating feature surfaces (API routes, frontend routes, tests, flags, docs) in common technology stacks. These are starting points, not absolute truths — always verify by reading the actual files before assuming a stack matches one of these patterns.

Use these patterns in two places:
1. **Bootstrap Step 2** — to locate feature surfaces in an unfamiliar project
2. **Drift-sync Step 2** — to extract the current set of routes/tests/flags and compare against the inventory

## API routes

The goal is to enumerate all HTTP routes the application exposes. Different frameworks register routes differently.

### FastAPI (Python)

- **Router registration**: Look for `app.include_router(...)` calls, typically in `main.py` or `app.py`. Each `include_router` call usually includes a `prefix="/api/v1/xxx"` argument.
- **Endpoint files**: Usually under `app/api/<version>/endpoints/` or `app/routes/`. Each file defines a `router = APIRouter()` and decorates handler functions.
- **Route decorators**: `@router.get("/path")`, `@router.post("/path")`, etc.
- **Runtime extraction** (most reliable): Import the FastAPI app and walk `app.routes`. Each route has `.methods` and `.path`. Requires the app to be importable, which may mean setting environment variables (e.g., a database URL) to dummy values so the app's startup code doesn't fail on missing config.
- **Static extraction** (fallback): `Grep` for `@router\.(get|post|put|patch|delete)` and parse the first argument.

Typical full path: `f"{router_prefix}{route_path}"`. You need to stitch them together.

### Express (Node.js)

- **Route files**: Usually under `routes/`, `src/routes/`, or inline in `app.js` / `server.js` / `index.js`.
- **Route registration patterns**:
  - `app.get('/path', handler)`
  - `router.post('/path', handler)`
  - `app.use('/prefix', subRouter)` — prefixes nested routers
- **No runtime introspection** without instrumentation — parse source.
- **Gotcha**: Prefix nesting via `app.use()` means you have to walk the call graph to reconstruct full paths.

### NestJS (Node.js)

- **Controllers**: Classes decorated with `@Controller('/path')`.
- **Route handlers**: Methods decorated with `@Get()`, `@Post()`, etc.
- **Full path**: Controller prefix + method path.
- **Module registration**: Routes are wired up in `app.module.ts` and submodules.

### Django (Python)

- **URL patterns**: `urls.py` files, both at the project root and per-app.
- **Patterns**: `path('projects/', views.projects, name='projects')`, `re_path(r'^projects/(?P<id>\d+)/$', ...)`
- **Nested includes**: `path('api/v1/', include('myapp.urls'))` — walk these recursively to build full paths.
- **Runtime extraction**: Use `django.urls.get_resolver().reverse_dict` or similar, but static parsing is usually simpler.

### Flask (Python)

- **Decorators**: `@app.route('/path')` or `@bp.route('/path')` with a Blueprint.
- **Blueprint registration**: `app.register_blueprint(bp, url_prefix='/api')`.
- **Runtime extraction**: `app.url_map.iter_rules()` gives you every registered route.

### Next.js (App Router — 13+)

- **File-based**: Each `app/<path>/route.ts` file is an endpoint.
- **HTTP methods**: Determined by exported function names (`export async function GET`, `POST`, etc.).
- **Path**: Derived from the directory structure. `app/api/projects/[id]/route.ts` → `/api/projects/:id`.
- **No runtime registry** — walk the filesystem.

### Next.js (Pages Router)

- **File-based**: `pages/api/*.ts`, each file is an endpoint.
- **All methods in one file**: Default export function handles all methods, typically with a `req.method` switch.

### Go (net/http, chi, gin, gorilla/mux)

- **net/http**: `http.HandleFunc("/path", handler)` or `mux.HandleFunc(...)`.
- **chi**: `r.Get("/path", handler)`, `r.Post(...)`, nested with `r.Route("/prefix", ...)`.
- **gin**: `r.GET("/path", handler)`, grouped with `r.Group("/api")`.
- **gorilla/mux**: `r.HandleFunc("/path", handler).Methods("GET")`.

### Ruby on Rails

- **Routes file**: `config/routes.rb`.
- **Patterns**: `resources :projects`, `get '/path', to: 'controller#action'`.
- **Runtime extraction**: `rails routes` command prints the full list.

## Frontend routes

### React Router

- **Definition files**: `App.tsx`, `routes.tsx`, `main.tsx`, or a dedicated `router.ts`.
- **Patterns**:
  - JSX style: `<Route path="/projects" element={<Projects/>} />`
  - Config style: `createBrowserRouter([{ path: '/projects', element: <Projects/> }])`
- **Nested routes**: A `<Route>` inside another `<Route>` or children in the config object.

### Next.js

- **App Router**: Each `app/<path>/page.tsx` is a route. Dynamic segments: `app/[id]/page.tsx`.
- **Pages Router**: Each file under `pages/` except `pages/api/` is a route.
- **No runtime registry** — walk the filesystem.

### Vue Router

- **Definition**: `src/router/index.ts` or `router.ts`, uses `createRouter({ routes: [...] })`.
- **Patterns**: `{ path: '/projects', component: Projects }`.

### SvelteKit

- **File-based**: Under `src/routes/`. Each `+page.svelte` is a route. Dynamic segments: `[id]`.

### Angular

- **Routing module**: `app-routing.module.ts`.
- **Patterns**: `const routes: Routes = [{ path: 'projects', component: ProjectsComponent }]`.

### Remix

- **File-based**: Under `app/routes/`. Flat file naming: `projects.$id.tsx`.

## Tests

Tests are conventional per-language but the actual location varies. Look for:

- **Python**: `tests/`, `test/`, `__tests__/`, `backend/tests/`. Check `pytest.ini`, `pyproject.toml`, or `setup.cfg` for the `testpaths` setting.
- **JavaScript / TypeScript**: `tests/`, `__tests__/`, co-located `*.test.ts` / `*.spec.ts`. Check `jest.config.js`, `vitest.config.ts`.
- **Go**: `*_test.go` files alongside the code. No central directory.
- **Rust**: `tests/` for integration tests, inline `#[cfg(test)]` modules for unit tests.
- **Java/Kotlin**: `src/test/java/`, `src/test/kotlin/`.
- **Ruby**: `spec/` (RSpec) or `test/` (minitest).

Smoke tests, E2E tests, and integration tests are often separated:
- `smoke_tests/`, `smoke/`
- `e2e/`, `integration/`
- `cypress/`, `playwright/`

If the inventory tracks `test_file`, `smoke_test`, or similar, clarify in `config.json` which category of test the field refers to.

## Feature flags

### Self-hosted flags

Common patterns:
- A Python module like `feature_flags.py` or `feature_gates.py` with constants or a function.
- A `feature_flags` database table with entries per flag.
- A TypeScript module exporting flag constants.
- Environment variables with a prefix like `FEATURE_` or `FLAG_`.

Search patterns:
- `Grep` for `feature_flag`, `featureFlag`, `feature_gate`, `isEnabled`, `FEATURE_`
- `Glob` for files named `*flag*`, `*gate*`

### Third-party flag services

If the project uses a flag service, flags often live outside the codebase. The inventory can still track them by name; drift-sync should skip the "flag exists in code" check for flags stored externally.

- **PostHog**: Look for `posthog-node`, `posthog-js`, or calls to `posthog.isFeatureEnabled('flag_name')`.
- **LaunchDarkly**: Calls to `ldClient.variation('flag-name', ...)`.
- **Unleash**: Calls to `unleash.isEnabled('feature-name')`.
- **Flagsmith**: Calls to `flagsmith.hasFeature('feature-name')`.
- **Flipper** (Ruby): `Flipper.enabled?(:feature_name)`.

Record the flag service in `config.stack.flag_service` so drift-sync knows to skip local file verification.

## Documentation

### Docusaurus

- Source files under `docs/` or `website/docs/`
- Markdown files with frontmatter (`id`, `title`, `slug`)
- Sidebar config in `sidebars.js` / `sidebars.ts`

### MkDocs

- Source files under `docs/`
- `mkdocs.yml` declares the nav structure

### Jekyll

- Source files under `_posts/` or `_docs/`
- `_config.yml` for configuration

### Hugo

- Source files under `content/`
- `hugo.toml` / `config.yaml`

### Custom / bespoke

- Look for a top-level `docs/` directory with markdown files
- Check README.md for links to a docs site
- Check package.json scripts for a `docs` command

## Important reminders

1. **Verify before assuming**. When bootstrap finds a candidate file, read it and confirm the pattern matches one of the above before treating it as canonical.
2. **Mixed stacks are common**. A project may have a FastAPI backend AND a Next.js frontend. Discover each separately.
3. **Multiple routers are common**. Large projects often have several API routers registered at different prefixes. Find all of them.
4. **Static parsing has limits**. Dynamic route registration (e.g., loop-based `for route in routes: app.add(route)`) may be invisible to a static scan. When in doubt, prefer runtime introspection if the framework supports it.
5. **Unknown stacks**. If the project uses a framework not listed here, apply first principles: find the entry point, follow the call chain, identify where HTTP handlers are wired up. Record what you learned in `runs.log` so future runs can start from your notes.
