# appsec.cc

A hands-on **dev-security training** (OWASP Top 10 Web + API) and the reproducible infrastructure that serves it online. Two complementary repos: the training **content** and its **deployment**.

🔗 Training server: **https://appsec.cc/**

---

## 📚 [`formation-dev-secu-java-vue`](https://github.com/appsec-cc/formation-dev-secu-java-vue) — the content

A hands-on **dev-security course** built around Java backends and Vue/Vite frontends. It's organized as 13 numbered case studies, each mapping to an **OWASP Top 10** (Web + API) category:

| # | Theme |
|---|-------|
| 01 | Broken Access Control / IDOR |
| 02 | Security Misconfiguration (Spring Boot, H2 console, Actuator) |
| 03 | Supply Chain Failures (Nexus npm registry, Vite, exfil listener) |
| 04 | Cryptographic Failures |
| 05 | Injection / XSS |
| 06–10 | API: BOLA, Broken Auth, BOPLA, Resource Consumption, BFLA |
| 11–13 | Combined 2021/2025 OWASP categories |

Each project typically runs on `localhost:8080` (embedded Tomcat or Spring Boot, in-memory H2), so they're meant to be launched **one at a time**. Project 03 is the exception (Nexus on 8081, Vite on 5173). The README also captures practical environment notes (WSL DNS/PTR lookup fix, switching JDKs on Windows).

## ⚙️ [`conf-as-code`](https://github.com/appsec-cc/conf-as-code) — the deployment / infra

An **Ansible-based provisioning repo** that turns those projects into a live training server reachable at `https://appsec.cc/`. Two-phase logic:

1. **`apps`** — rsync sources from *both* repos: the formation repo (the `NN-*` projects + their READMEs) and this repo's `front/` (a Next.js static doc site), then pre-build everything (Maven with the right `JAVA_HOME`, Node builds), and install systemd units (`appsec-app@…`).
2. **`switch`** — flip the single active project: stop current → start new → swap the nginx config. Run with:
   ```bash
   ansible-playbook -i inventory.ini playbook.yml --tags switch -e project=XX
   ```

**Roles**: `common` (Java 8+17, Maven, Node 20, UFW), `nginx` (reverse proxy + TLS via certbot/Let's Encrypt), `nexus` (bare-metal Nexus for project 03), `apps`, `switch`. UFW whitelists trainee IPs.

## 🔗 The relationship between the two

- **`formation-dev-secu-java-vue`** = the **vulnerable apps + course material** (the AppSec course content).
- **`conf-as-code`** = the **infrastructure** that builds, serves, and hot-swaps them behind nginx/TLS on a shared server.

The two cross-reference each other (`formation_repo` is a sibling dir by default), and the doc site in `conf-as-code/front` reads the projects' `README.md` files via `APPSEC_CONTENT_DIR`.

# appsec-cc.github.io

> **Status: in progress.** The `basePath` change (`conf-as-code/front`), the build workflow (`appsec-cc.github.io/.github/workflows/snapshot.yml`), and the `SOURCE_REPOS_PAT` secret are done. **Remaining (manual, in GitHub UI):** set Pages → Source = GitHub Actions, then run the workflow.

## Why

The live training server (`https://appsec.cc/`) is being **temporarily shut down** to save hosting cost while no session is running. When training is scheduled again, `conf-as-code` will redeploy the exact same stack — including `conf-as-code/front` **unchanged** — to a fresh server.

In the meantime, the course material should stay **browsable online without any running server**. The solution is to publish a **static snapshot** of the doc site to GitHub Pages at `https://appsec-cc.github.io/`.

Key principles:

- **`conf-as-code/front` remains the single source of truth.** It is reused as-is for the next server deploy. Nothing is migrated.
- **`appsec-cc.github.io` contains only the build workflow — no generated files are ever committed.** The site is built and deployed straight to Pages as an artifact, so there is nothing to hand-edit and nothing to drift. If the training changes, re-run the workflow.
- The Next.js app already uses `output: 'export'` and reads the project `README.md` files **at build time**, so the resulting `out/` is fully self-contained (no Node server, no `APPSEC_CONTENT_DIR` at serve time).

## The one constraint: `basePath`

`conf-as-code/front/next.config.js` is hardwired to `basePath: '/docs'` because the live site serves at `appsec.cc/docs`. But `appsec-cc.github.io` is an **org Pages site served at the root** (`https://appsec-cc.github.io/`, no `/docs` prefix). Building with `basePath: '/docs'` would 404 every asset and link.

Since `front/` must stay reusable as-is, **make `basePath` env-driven** rather than hardcoding the new value:

```js
// conf-as-code/front/next.config.js  — implemented
const nextConfig = {
  output: 'export',
  basePath: process.env.APPSEC_BASE_PATH ?? '/docs', // default unchanged → server deploy unaffected
  trailingSlash: true,
  images: { unoptimized: true },
}
```

The server deploy keeps the `/docs` default; the snapshot build overrides it to `''` (root).

## How it is published: a GitHub Action in `appsec-cc.github.io`

The snapshot is built and deployed by a **GitHub Action that lives in the `appsec-cc.github.io` repo**, using the native **"GitHub Actions" Pages source** (not "deploy from branch").

Why this repo and this model:

- GitHub Pages deployment is **repo-bound** — the `actions/upload-pages-artifact` → `actions/deploy-pages` flow publishes to the Pages site of the repo it runs in. To deploy `appsec-cc.github.io`'s Pages with no cross-repo token, the workflow must live there.
- Because the built site is **uploaded as an artifact and deployed directly**, no generated files are ever committed. The repo holds **only the workflow** — which is exactly what keeps the snapshot from being hand-edited or drifting.

The workflow checks out the two source repos as **build inputs**, builds at root, and deploys:

Committed at `appsec-cc.github.io/.github/workflows/snapshot.yml`:

```yaml
name: Publish snapshot
on:
  workflow_dispatch:        # trigger manually when a refreshed snapshot is wanted
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deploy.outputs.page_url }}
    steps:
      - name: Checkout doc-site source (front/)
        uses: actions/checkout@v4
        with:
          repository: appsec-cc/conf-as-code
          path: conf-as-code
          token: ${{ secrets.SOURCE_REPOS_PAT }}   # private repo → needs read token
      - name: Checkout course content (README.md per project)
        uses: actions/checkout@v4
        with:
          repository: appsec-cc/formation-dev-secu-java-vue
          path: formation-dev-secu-java-vue
          token: ${{ secrets.SOURCE_REPOS_PAT }}   # private repo → needs read token
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: conf-as-code/front/package-lock.json
      - name: Build static export at root
        working-directory: conf-as-code/front
        env:
          APPSEC_BASE_PATH: ''                                      # serve at root, not /docs
          APPSEC_CONTENT_DIR: ${{ github.workspace }}/formation-dev-secu-java-vue
        run: |
          npm ci
          npm run build       # → conf-as-code/front/out/
      - uses: actions/upload-pages-artifact@v3
        with:
          path: conf-as-code/front/out
      - name: Deploy to GitHub Pages
        id: deploy
        uses: actions/deploy-pages@v4
```

> Note: both source repos (`conf-as-code`, `formation-dev-secu-java-vue`) are **private**, so the two `checkout` steps **must** authenticate with a token that has read access to them — the default `GITHUB_TOKEN` is scoped to `appsec-cc.github.io` only. Create a **fine-grained PAT** with read-only `Contents` access to both repos and store it as the `SOURCE_REPOS_PAT` secret in `appsec-cc.github.io`.

**When the live server returns**, nothing has to be undone: `conf-as-code/front` was never touched, so the server deploy is unaffected. Re-run the workflow to refresh the snapshot, or disable Pages to retire it.

## TODO checklist

- [x] Make `basePath` env-driven in `conf-as-code/front/next.config.js` (`APPSEC_BASE_PATH`, default `/docs`)
- [x] Add `.github/workflows/snapshot.yml` to `appsec-cc.github.io` (checks out `conf-as-code` + `formation-dev-secu-java-vue`)
- [x] Create a fine-grained read-only PAT (Contents) for both private source repos and store it as the `SOURCE_REPOS_PAT` secret in `appsec-cc.github.io`
- [ ] In `appsec-cc.github.io` settings, set **Pages → Source = GitHub Actions**
- [ ] Run the workflow and confirm `https://appsec-cc.github.io/` serves with no broken asset/link paths (this also verifies the root build renders with no broken asset/link paths)