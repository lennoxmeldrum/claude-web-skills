---
name: aistudio2gcr
description: Convert a freshly-pushed Google AI Studio repository into a Cloud Run-ready app. Adds a Dockerfile, nginx config with iframe-friendly headers, runtime environment-variable injection (docker-entrypoint.sh + runtime-config.js), an API_KEY fallback alongside GEMINI_API_KEY, verifies the index.html entry-point script tag, and renames the default AI Studio page title. Use whenever the user asks to prepare an AI Studio repo for Cloud Run, mentions GCR/Cloud Run deployment of a Vite/React app from AI Studio, or pastes/attaches such a repo with that intent.
---

# AI Studio → Cloud Run conversion

The user has just created an app in Google AI Studio and pushed it to GitHub. They will be connecting the repo to Cloud Run. Look at the code and add everything needed to make that connection deploy cleanly. Work on the user's currently-attached repo (or the one they specify); make all changes on a feature branch and open a PR when finished.

## Required changes

### 1. Dockerfile + build tooling
Add a Dockerfile suitable for a Vite static build served by nginx. If using `npm ci`, **never forget to also commit the `package-lock.json` file** — generate one (`npm install` once locally inside the Dockerfile build context, or instruct the user) if it's missing. This has been a recurring miss in the past.

### 2. API key configuration
If the code references `GEMINI_API_KEY` (or any similar specific API key name), add a fallback to a generic `API_KEY` environment variable using the pattern:

`process.env.GEMINI_API_KEY || process.env.API_KEY`

This way the user only needs to set `API_KEY` in Cloud Run secrets, but the original code references are preserved. Apply the same fallback in `vite.config.ts` `define` block if it injects `process.env.GEMINI_API_KEY`.

### 3. AI Studio entry point
AI Studio apps require a `<script type="module">` tag in `index.html` that loads the application entry point. Verify that the script tag points at the **actual entry file in this repo** — it might be `/index.tsx`, `/src/main.tsx`, `/src/index.tsx`, or similar. Open the repo and confirm the path matches a real file. Without a correct script tag, Vite won't bundle the application code and the page will be blank.

### 4. Page title
Replace the default `<title>` in `index.html` (AI Studio leaves it as "My Google AI Studio App" or similar) with a concise, descriptive name based on the repository name and the app's purpose.

### 5. Runtime environment variables
For static builds deployed to Cloud Run, implement runtime environment variable injection:

- Create `docker-entrypoint.sh` that generates `runtime-config.js` from Cloud Run env vars at container start.
- Update `index.html` to load `runtime-config.js` **before** app initialization.
- Update API service files to check `window.RUNTIME_CONFIG` before `process.env`.
- Add `public/runtime-config.js` as a placeholder for local development.

### 6. nginx caching
Ensure nginx **never caches** any configuration file such as `runtime-config.js`. Add an explicit `location` block disabling caching for those files.

### 7. Iframe-friendly headers (for Google Sites embedding)
The user needs to embed the deployed URL in a Google Site. If using nginx in production, update `nginx.conf` to remove `X-Frame-Options` entirely and use `Content-Security-Policy` instead. Use exactly this header block:

    # Security headers - Allow embedding in iframes
    # X-Frame-Options removed - using CSP instead
    add_header Content-Security-Policy "frame-ancestors *" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

If there's no nginx (e.g. a Node server), ensure the production server doesn't set restrictive frame headers.

## Workflow

1. Read the repo to confirm: framework (assume Vite/React unless evidence otherwise), entry-point file, API key references, current `index.html` `<title>`, presence of `package-lock.json`.
2. Make all the changes above on a feature branch.
3. Commit with clear messages and push.
4. Open a PR (ready for review, not draft).
5. In the final summary, tell the user:
   - Which secrets to set in Cloud Run (`API_KEY` at minimum).
   - The expected Cloud Run URL pattern and the iframe snippet to drop into Google Sites: `<iframe src="URL" width="100%" height="100%"></iframe>`.
   - Anything they need to do manually in the Cloud Run console.
