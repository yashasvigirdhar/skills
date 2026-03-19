# Chrome DevTools MCP — Gotchas

Common pitfalls when running E2E browser tests via Chrome DevTools MCP. These were discovered across hundreds of nightly QA runs.

## Cookie injection

- On domains like `lvh.me`, set cookies with `path=/` only. Adding a `domain=` parameter can cause silent failure — cookies are not stored. Omit the domain parameter entirely when in doubt.
- Auth flows (like signup) may set their own session/CSRF cookies. When switching to a different session, the old cookies persist alongside the new ones, causing auth errors. Always clear old cookies across all domain/path combos before injecting new session cookies.

## React controlled inputs

The Chrome DevTools `fill` tool sets the DOM value but does NOT trigger React's `onChange` for controlled components. Buttons that depend on state (like "Save") stay disabled.

**Workaround 1 — native event dispatch** (works for most inputs):
```javascript
const setter = Object.getOwnPropertyDescriptor(
  window.HTMLTextAreaElement.prototype, 'value'  // or HTMLInputElement
).set;
setter.call(element, 'your text');
element.dispatchEvent(new Event('input', { bubbles: true }));
element.dispatchEvent(new Event('change', { bubbles: true }));
```

**Workaround 2 — React internal onChange** (when native events don't work):
```javascript
const reactPropsKey = Object.keys(element).find(k => k.startsWith('__reactProps'));
element[reactPropsKey].onChange({ target: { value: 'your text' } });
```

## Session management

- Auth sessions can expire mid-run during long test suites. If a navigation unexpectedly redirects to the login page, re-create the session and re-inject cookies.
- When switching between user roles (e.g., admin → regular user), always clear ALL auth-related cookies before injecting the new session. Leftover cookies from the previous role cause subtle auth failures.

## Navigation

- Clicking tab elements doesn't always trigger client-side navigation. If the URL doesn't change after clicking a tab, navigate directly via URL instead.
- Some actions (like "Preview") open content in a new browser tab. Use `list_pages` to discover the new tab, `select_page` to switch to it, then `close_page` when done.

## Context window management

- Long runs (20+ tests + cleanup) can exceed context limits. Write results to the report after EVERY test, not in batches. This prevents data loss when context compacts.
- When uncertain about what you've already tested after context compression, read the report file — it's the source of truth.

## Database cleanup

- When using `psql` in batch mode, a single `-c` call with multiple statements uses an implicit transaction that rolls back ALL statements if ANY one fails. For cleanup reliability, either use separate `psql -c` calls per statement or set `ON_ERROR_ROLLBACK=on`.
- Always delete child records before parent records (respect FK order). Map out the FK chain for your schema before writing cleanup queries.
