---
name: cloudflare-ready
description: Make an app behave correctly behind the ExplAIn Sims Cloudflare Worker, which path-strips the first URL segment (so `explainsims.com/<app>/foo` reaches Cloud Run as `/foo` even though the browser still shows `/<app>/foo`). The Worker's HTMLRewriter only prepends the prefix on root-absolute values of `script@src`, `link@href`, `img@src`, and `a@href`; everything else (form actions, fetch/XHR/axios URLs, `window.location` assignments, OAuth/Firebase redirect URIs, JS-injected DOM, React Router `Link` targets) leaks through unchanged and breaks. Use whenever a deployed Cloud Run app reachable via `explainsims.com/<app>` renders the first page correctly but breaks on a form submission, client-side navigation, an OAuth round-trip, or an in-page fetch — typically symptomised by the URL bar dropping the `/<app>/` prefix on the next request.
---

# Cloudflare-Worker-ready URL handling

The user has an app deployed to Cloud Run behind the ExplAIn Sims Cloudflare Worker (one shared Worker handles every app). The Worker:

1. Splits the request path. The first non-empty segment is the app name; the rest is the asset path.
2. For non-Next.js apps, fetches `https://<app>-<project>.us-west1.run.app/<assetPath>` — so **the Cloud Run app is always served at root** even though the browser sees `/<app>/...`.
3. Pipes any HTML response through an `HTMLRewriter` that prepends `/<app>` to root-absolute attribute values, but **only on these four attributes**: `script@src`, `link@href`, `img@src`, `a@href`. The rewriter only acts when the value starts with `/` and not `//`, and skips values already prefixed with `/<app>/`.

Anything else with a root-absolute `/foo` path reaches the browser unmodified, so the next request fires off to `explainsims.com/foo` — bypassing the app's prefix and either 404-ing or being routed to a different app entirely. The Worker's Referer-based "ghost catcher" rescues some of these (mostly same-origin `fetch` calls) but it's fragile and doesn't keep the address bar under `/<app>/`, which cascades into broken sub-asset requests.

## Things that break (the audit checklist)

For each app, search the codebase and fix every occurrence of:

1. **`<form action="/...">`** — `action` is **not** in the rewriter's attribute list. Forms post to root.
2. **`fetch('/...')`, `axios.get('/...')`, `XMLHttpRequest` with `/...`** — JS-issued requests are never rewritten.
3. **`window.location.href = '/...'`, `window.location.assign('/...')`, `window.location.replace('/...')`, `history.pushState(..., '/...')`** — same story.
4. **OAuth / Firebase / magic-link redirect URIs** — anywhere code constructs `window.location.origin + '/some/hardcoded/path'`, the hardcoded path drops the prefix. (This is the bug that hit `fiba-2027`'s Firebase magic-link flow.) Use `window.location.origin + window.location.pathname` instead.
5. **React Router `<Link to="/...">` / `<Navigate to="/...">` / `useNavigate()(...)` / `createBrowserRouter` route paths** — these emit DOM after the initial HTML has gone through the rewriter, so they keep their root-absolute form. Set the router `basename` to the runtime-derived prefix.
6. **JS-injected DOM**: `el.setAttribute('href', '/foo')`, `new Image().src = '/foo'`, dynamic `<script>` tags, `el.innerHTML = '<a href="/foo">…'` — none of these go through the rewriter.
7. **CSS `url(/foo)`, `manifest.json` `start_url`/`scope`, sitemap, robots.txt** referencing `/foo` paths — never rewritten.

## The fix pattern

Derive the prefix at runtime from the first path segment, and prepend it everywhere it's missing:

```js
const APP_PREFIX = '/' + (window.location.pathname.split('/').filter(Boolean)[0] || '');
// Behind the Worker → '/<app>'.  In local dev (app at root) → '/' degenerates to ''.
```

Apply this to the broken sites:

- **Forms**: drop the `action` attribute from HTML and set it via JS:
  ```js
  document.getElementById('loginForm').action = APP_PREFIX + '/start';
  ```
- **fetch / axios / XHR**: prepend the prefix:
  ```js
  fetch(APP_PREFIX + '/api-proxy/...');
  ```
- **`window.location.href`**: same:
  ```js
  window.location.href = APP_PREFIX + '/';
  ```
- **OAuth / Firebase magic-link redirect**: use `window.location.pathname` (already includes the prefix) for the redirect target:
  ```js
  const actionCodeSettings = {
    url: window.location.origin + window.location.pathname,
    handleCodeInApp: true,
  };
  ```
- **React Router**: pass the runtime prefix as `basename`:
  ```jsx
  const basename = '/' + (window.location.pathname.split('/').filter(Boolean)[0] || '');
  <BrowserRouter basename={basename}>...</BrowserRouter>
  ```
  (or equivalent for `createBrowserRouter`.)

## What you should NOT change

- **`<a href="/...">`, `<script src="/...">`, `<link href="/...">`, `<img src="/...">`** in static/server-rendered HTML: leave them as root-absolute. The Worker's HTMLRewriter prepends the prefix for you. Do not double-prefix — the rewriter's "skip if already starts with `/<app>/`" guard means a manually-prefixed value will work, but a runtime-derived one will be wrong in local dev where there is no prefix.
- **Vite `base`**: leave it at the default `/`. Setting `base: '/<app>/'` hardcodes the prefix into the built `index.html` and into Vite's emitted asset URLs, which then have to be stripped again — the rewriter handles asset prefixing for you, so Vite doesn't need to.
- **Next.js apps**: this skill does not apply. Next.js apps belong in the Worker's `nextJsApps` allow-list and use a Next.js `basePath` config; their full path is preserved end-to-end by the Worker.

## Workflow

1. Confirm the app is in fact behind the Worker (Cloud Run app reachable via `explainsims.com/<app>/`) and is **not** in the Worker's `nextJsApps` list.
2. Grep the repo thoroughly for the broken patterns above. At minimum: `action="/`, `fetch("/`, ` fetch('/`, `` fetch(`/ ``, `location.href = "/`, `location.href = '/`, `location.assign`, `location.replace`, `history.pushState`, `redirect_uri`, `actionCodeSettings`, `<Link to="/`, `navigate("/`, `navigate('/`. JSX attributes count.
3. For each hit, decide: runtime-prefix? truly relative path (no leading slash)? Or `window.location.pathname`-based? Pick the simplest correct option. (Truly relative paths have a trailing-slash gotcha — `start` resolves to `/start` when the document URL is `/<app>` with no trailing slash. Runtime-prefix is safer.)
4. Apply fixes on a feature branch, commit with a clear message, push, and open a PR ready for review.
5. **Test in production by visiting `https://explainsims.com/<app>/` and clicking through every flow that previously broke.** The URL bar should stay under `/<app>/` the whole time. If after a navigation, form submission, or auth round-trip the URL drops the prefix, you missed a spot.

## Worker reference

For the full Worker behaviour (lowercase canonicaliser, Referer ghost-catcher, HTMLRewriter scope, Next.js routing branch, project-number overrides), read the deployed Worker source. The exact rewritten attribute set lives in:

```
.on("script", new AttributeRewriter("src", appName))
.on("link",   new AttributeRewriter("href", appName))
.on("img",    new AttributeRewriter("src", appName))
.on("a",      new AttributeRewriter("href", appName))
```

Anything not on this list is the app's responsibility.
