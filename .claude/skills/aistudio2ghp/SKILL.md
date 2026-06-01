---
name: aistudio2ghp
description: Convert a freshly-pushed Google AI Studio repository into a GitHub Pages-deployable app. Sets the Vite base path, adds a .nojekyll file, verifies the index.html entry-point script tag, renames the default AI Studio page title, adds a fitting inline-SVG emoji favicon, ensures iframe-embeddability, strips root-absolute leading slashes for Cloudflare compatibility, and adds a self-configuring GitHub Actions workflow that builds and deploys to GitHub Pages automatically on push to main. Use only for apps that do NOT require runtime API keys. Optionally, after the GH Pages conversion, also reimplements the app as a native vanilla HTML/CSS/JS page in the ExplAIn Sims site (explainsims/explainsims.github.io) so it matches the rest of the site's look and conventions — this is a hand-ported parallel rebuild, NOT an iframe wrapper, and only happens on explicit user request. Use whenever the user asks to prepare an AI Studio repo for GitHub Pages.
---

# AI Studio → GitHub Pages conversion

The user has just created an app in Google AI Studio and pushed it to GitHub. They want it deployed as a standalone GitHub Pages site that is also embeddable as an iframe. This skill assumes the app does NOT require a runtime API key — that's a deliberate constraint the user has set, since GH Pages can't safely hold one.

Work on the user's currently-attached repo (or the one they specify); commit all changes directly to the main branch to instantly trigger the automatic deployment pipeline.

## Part 1 — GitHub Pages conversion (always do this)

### 1. Vite base path
In `vite.config.ts`, set the `base` option so assets resolve under the GitHub Pages subpath:

- For a project page at `https://<owner>.github.io/<repo>/`: `base: '/<repo>/'`
- For a user/org page at `https://<name>.github.io/`: `base: '/'`

Read the repo name from the remote and use it; do not guess.

### 2. `.nojekyll`
Add an empty `.nojekyll` file at the repo root so GitHub Pages serves files starting with `_`.

### 3. AI Studio entry point
AI Studio apps require a `<script type="module">` tag in `index.html` that loads the application entry point. Verify that the script tag points at the actual entry file in this repo — it might be `/index.tsx`, `/src/main.tsx`, `/src/index.tsx`, or similar. Open the repo and confirm the path matches a real file. Without a correct script tag, Vite won't bundle the application code and the page will be blank.

### 4. Page title & favicon
Replace the default `<title>` in `index.html` (AI Studio leaves it as "My Google AI Studio App" or similar) with a concise, descriptive name based on the repository name and the app's purpose.

While you're in the `<head>`, give the app a favicon so the browser tab shows a real icon instead of the blank default — AI Studio apps ship without one. Use an **inline SVG data-URI** holding a single emoji glyph chosen to fit what the app actually does (e.g. 🧭 for a navigation/trip tool, ⚛️ for a physics sim, 📊 for a data dashboard, 🗺️ for a map). Add it right after the `<title>`:

```html
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🧭</text></svg>" />
```

Pick the emoji to match the subject; don't leave the placeholder compass unless the app is actually about navigation. Prefer this inline data-URI over a separate `favicon.ico`/`.png` file because:
- It needs no extra files and no build/asset wiring (and nothing to add to the Vite `base`/`public` plumbing).
- It is immune to the Cloudflare subpath proxy — a data-URI has no path to rewrite, so it resolves identically under `explainsims.com/<app>/`, the direct `*.github.io/<repo>/` URL, and localhost. A root-absolute `/favicon.ico` would break under both the GH Pages subpath and the proxy (the same class of bug section 6 exists to prevent).

If the repo already ships a brand logo/icon you'd rather use, that's fine, but reference it so the path still resolves under the subpath (see section 6); when in doubt, the emoji data-URI is the safe default.

### 5. Iframe-friendliness
GitHub Pages does not add `X-Frame-Options` to user content, so iframes generally work. To be explicit and to satisfy stricter embedders (like Google Sites), add a meta CSP to `<head>` in `index.html`:

`<meta http-equiv="Content-Security-Policy" content="frame-ancestors *">`

Do not add an `X-Frame-Options` meta tag — it has no effect as a meta and only causes confusion.

### 6. Relative asset path correction (Cloudflare Subpath Compatibility)
The application will be served behind a Cloudflare Worker reverse proxy using a subpath prefix which may or may not be accessed with a trailing slash.- Scan all source files (especially `.tsx`, `.ts`, `.jsx`, `.js`, and `index.html`) for root-absolute asset paths.
- Define a global dynamic path-resolver helper within the main entry files (such as App.tsx or a shared utility file):

```typescript
const getAssetPath = (url: string) => {
  const clean = url.startsWith('/') ? url.slice(1) : url;
  const isProxy = window.location.hostname.includes('explainsims.com');
  const segments = window.location.pathname.split('/').filter(Boolean);
  return isProxy && segments.length > 0 ? '/' + segments[0] + '/' + clean : '/' + clean;
};
```

- Scan all source code files (such as .tsx, .ts, .jsx, .js, and index.html) for static root-absolute or standard relative asset paths (e.g., /CASEL.png or logo.png).
- Wrap and resolve these paths utilizing the dynamic helper. For example, replace `<image href="/CASEL.png" />` with `<image href={getAssetPath("CASEL.png")} />`. This guarantees correct resolution under both explainsims.com subpath contexts (with or without trailing slashes) and direct static *.github.io / localhost URLs.

### 7. GitHub Actions deployment workflow
Create a file named `.github/workflows/deploy-pages.yml`. This file must configure an automatic workflow to build and deploy the application. The workflow must explicitly define the `github-pages` environment to bypass manual configuration screens. Write it exactly like this template:

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: 'pages'
  cancel-in-progress: false

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Dependencies
        run: |
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install && npm shrinkwrap
          fi

      - name: Build Application
        run: npm run build

      - name: Configure Pages Artifact
        uses: actions/configure-pages@v5

      - name: Upload Build Output
        uses: actions/upload-pages-artifact@v3
        with:
          path: './dist'

      - name: Deploy Live to Pages
        id: deployment
        uses: actions/deploy-pages@v4

```
### 8. Sanity check & report
Confirm `package.json` has a `build` script that produces `dist/`. In the final conversation summary, tell the user:

- The exact GH Pages URL the site will live at.
- The iframe snippet for embedding: `<iframe src="URL" width="100%" height="100%"></iframe>`.

## Part 2 — ExplAIn Sims integration (only on explicit user request)

After Part 1 is done, ask the user whether they want to also integrate this app into the ExplAIn Sims site (explainsims/explainsims.github.io). Frame the question as optional. If they say no or don't ask for it, stop after Part 1.

This integration is not an iframe wrapper. The AI Studio app (typically React + Vite + TypeScript) must be rewritten as a native ExplAIn Sims page using vanilla HTML/CSS/JS so it matches the look, feel, and conventions of the rest of the site. The standalone GH Pages build from Part 1 stays as the source-of-truth for the original implementation; the ExplAIn Sims version is a parallel, hand-ported rebuild.

If they say yes, gather these details from the user before starting (use AskUserQuestion):

1. Tab — which tab page does this belong on? (appcm.html for AP PCM, tools.html for Tools, fun.html for Fun, panphy.html for PanPhy, or another.)
2. Unit / section — for tab pages that are organised by unit (e.g. appcm.html), which unit/section should the card go into?
3. App slug — the snake_case slug for the page file (e.g. friction_lab); default to a slug derived from the repo name and confirm.
4. Card copy — short title and one-line blurb for the card.
5. Featured? — should the app also be added to the FEATURED_POOL array in index.html for the featured rotation?

Then perform the integration in the explainsims/explainsims.github.io repo:

### A. Read the conventions first
Read explainsims/explainsims.github.io/CLAUDE.md and open several existing sims in the same tab directory before writing any code. Match the established patterns exactly — banner markup, help panel structure, theme variables and tokens, footer wiring, layout primitives, control panel styling. Do not improvise; copy from neighboring pages.

### B. Port the app to vanilla HTML/CSS/JS at <tab-dir>/<slug>.html
Create a self-contained page (e.g. appcm/<slug>.html) that reimplements the AI Studio app from scratch:

- No build step, no iframe, no React/JSX, no TypeScript, no bundler. Plain HTML, plain CSS, and plain JS in <script> tags or co-located .js files following whatever the rest of that tab directory uses. If neighboring sims use ES modules from a CDN (e.g. for math or plotting), follow that pattern.
- Read the source of the AI Studio app (typically App.tsx, index.tsx, components/*.tsx%, plus any state hooks/contexts) and translate the logic, state, math, and rendering into vanilla DOM/Canvas/SVG. Preserve the actual behaviour, equations, and parameter ranges of the simulation — do not paraphrase physics or simplify the model. If something is genuinely impractical to port 1:1, flag it explicitly rather than silently dropping it.
- Reuse the site's design tokens (CSS variables for colors, spacing, typography, shadows) instead of carrying over AI Studio's bespoke styling. The ported page must visually belong on ExplAIn Sims, not look like a transplanted React app.
- Include the standard ExplAIn Sims chrome by copying from a neighboring sim:
  - Sticky banner with home logo, back button, page title, theme toggle.
  - Per-app localStorage theme key: <slug>-dark (and <slug>-light if the convention uses both).
  - Help panel describing what the sim does and its controls, populated from the AI Studio app's UX/instructions.
  - Shared footer (<div id="site-footer"></div> + <script src="/assets/footer.js"></script> immediately before </body>).
  - Brave iOS gradient-text fix script if surrounding pages use it.
- Layout uses the same flex/grid primitives as neighboring sims; do not hardcode pixel heights for the main content area.

If the AI Studio app pulls in third-party libraries (e.g. a charting lib, a physics engine, KaTeX), prefer the same CDN-loaded vanilla version that other ExplAIn Sims pages already use. If no equivalent is already in use on the site, reimplement the needed subset directly rather than introducing a new dependency just for this page.

### C. Card on the tab page
Add a card on the tab HTML (e.g. appcm.html) under the correct unit/section:

- card-source-pill text matching the tab name ("AP PCM" for appcm, "Tools" for tools, "Fun" for fun, etc.).
- Title and blurb from the user's input.
- Link points to /<tab-dir>/<slug>.html.
- Match the markup of surrounding cards in that section exactly.

### D. Sitemap & featured pool
- Add /<tab-dir>/<slug>.html to sitemap.xml.
- If the user said yes to featured, add an entry to FEATURED_POOL in index.html.

### E. Branch + PR
Use a claude/ branch prefix per ExplAIn Sims conventions. Open a PR ready for review (not draft).

### F. Verification
Open the new page in a browser (or the local dev server the repo uses) and exercise the simulation alongside the original AI Studio / GH Pages version. Confirm parity on the main interactions, that the theme toggle works, and that the help panel and footer render correctly. If the port has known gaps vs. the original, list them in the PR description.

## Workflow

1. Read the target application repo to confirm framework setup, entry point, title, and file paths.
2. Generate/edit all required files directly on the main branch. 
3. Commit and push the changes directly to GitHub. 
4. In the final summary conversation, provide the expected live URL pattern based on the organization name and repository name, alongside a clean <iframe src="URL" width="100%" height="100%"></iframe> embedding block. Remind the user that the background Pages builder is running and the link will go live in 1-2 minutes.
