---
name: aistudio2gcr
description: Convert a freshly-pushed Google AI Studio repository into a Cloud Run-ready app. Adds a Dockerfile, nginx config with iframe-friendly headers, runtime environment-variable injection (docker-entrypoint.sh + runtime-config.js), an API_KEY fallback alongside GEMINI_API_KEY, verifies the index.html entry-point script tag, renames the default AI Studio page title, and generates a GitHub Actions workflow for zero-click automatic deployment. Use whenever the user asks to prepare an AI Studio repo for Cloud Run, mentions GCR/Cloud Run deployment of a Vite/React app from AI Studio, or pastes/attaches such a repo with that intent.
---

# AI Studio → Cloud Run conversion

The user has just created an app in Google AI Studio and pushed it to GitHub. They will be connecting the repo to Cloud Run via automated GitHub Actions. Look at the code and add everything needed to make that connection deploy cleanly. Work on the user's currently-attached app repo; commit all changes directly to the main branch to instantly trigger the deployment.

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

### 8. Automated GitHub Actions Deployment File
Create a file named `.github/workflows/deploy.yml`. This file must configure an automatic workflow that fires on every push to the main branch. Write it exactly like this template so it pulls settings dynamically from the repository metadata and organization variables:

```yaml
name: Automated Cloud Run Deploy
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}

    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v2

    - name: Configure Docker for GCP
      run: gcloud auth configure-docker us-west1-docker.pkg.dev --quiet

    - name: Build and Push Container
      run: |
        IMAGE_NAME="us-west1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/cloud-run-source-deploy/${{ github.event.repository.name }}:latest"
        docker build -t $IMAGE_NAME .
        docker push $IMAGE_NAME

    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy ${{ github.event.repository.name }} \
          --image us-west1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/cloud-run-source-deploy/${{ github.event.repository.name }}:latest \
          --region ${{ vars.GCP_REGION || 'us-west1' }} \
          --allow-unauthenticated


```

### 9. Relative asset path correction (Cloudflare Subpath Compatibility)
The application will be served behind a Cloudflare Worker reverse proxy using a subpath prefix (e.g., domain.com/repository-name/) which may or may not be accessed with a trailing slash.
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
- Wrap and resolve these paths utilizing the dynamic helper. For example, replace `<image href="/CASEL.png" />` with `<image href={getAssetPath("CASEL.png")} />`. This guarantees correct resolution under both explainsims.com subpath contexts (with or without trailing slashes) and direct *.run.app / localhost URLs.

## Workflow

1. Read the target application repo to confirm framework setup, entry point, title, and file paths.
2. Generate/edit all required files directly on the `main` branch. 
3. Commit and push the changes directly to GitHub. 
4. In the final summary conversation, provide the expected live URL pattern based on the project ID and repository name, alongside a clean `<iframe src="URL" width="100%" height="100%"></iframe>` embedding block. Remind the user that the background action trigger is running and the link will go live in 2-3 minutes.
