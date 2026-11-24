# Modernize deploy + surface resume.levijcl.com — Design

## Intent
The current flow (build `src/output.css` locally, commit it, Pages serves `main` as-is) works but feels dated. Move to a modern GitHub Actions deploy, make the README useful, and make the new domain `resume.levijcl.com` visible everywhere.

## Goals
- Push to `main` → Actions builds Tailwind and deploys to Pages automatically.
- Build output no longer tracked in git.
- README shows the live site (new domain) and documents dev + deploy.
- Pages and shared links point to `resume.levijcl.com`.

## Non-goals / anti-goals
- No Vite or other bundler, and no framework. The site stays two static HTML pages.
- No changes to resume content or layout. Print styles must keep working.
- The site must not go down (unstyled or 404) during the switch.

## Approaches
- **A. Actions build → `upload-pages-artifact` → `deploy-pages`** (picked): the officially recommended path, with the fewest changes.
- B. Actions commit `output.css` back, or push to `gh-pages`: no settings change, but bot commits pile up in history and the approach is dated.
- C. Vite + A: gives a dev server, but it's overkill for two pages.

## Design
1. **Workflow** `.github/workflows/deploy.yml`
   - Triggers: push to `main` and `workflow_dispatch`.
   - Build job: checkout → setup-node (Node 22, npm cache) → `npm ci` → `npm run build` → stage `index.html`, `cv/`, `src/output.css`, `CNAME` into `_site/` → `upload-pages-artifact`.
   - Deploy job: `deploy-pages`, with the `pages: write` and `id-token: write` permissions and the `github-pages` environment.
2. **package.json scripts**: `dev` (tailwind `--watch`) and `build` (tailwind `--minify`).
3. **Remove `src/output.css`** from git and add it to `.gitignore`.
4. **Pages setting**: switch `build_type` from `legacy` to `workflow` via `gh api`, **before** merging. Until the new workflow deploys, the existing deployment stays live, so nothing goes down. The custom domain and HTTPS stay configured.
5. **Domain links**: `index.html:38` and `cv/index.html:24` currently display `levijcl.github.io/...`. Change the displayed text to `resume.levijcl.com/cv` and `resume.levijcl.com`. The relative hrefs stay.
6. **Meta tags** on both pages: `description` (based on the Summary), `canonical`, `og:title`, `og:description`, `og:url`, and `og:type`. No og:image, since none exists.
7. **README**: title and one-line intro, live links (Resume / CV) on the new domain, a deploy-status badge, the tech used (HTML + Tailwind v4 + Pages via Actions), local dev (`npm ci`, `npm run dev`, then open `index.html`), and how deploy works.
8. **Repo About**: set the homepage to `https://resume.levijcl.com` via `gh repo edit`.

## Left to the builder
- Exact README wording and the meta description wording.
- Action versions: use the current major versions of `actions/checkout`, `setup-node`, `configure-pages`, `upload-pages-artifact`, and `deploy-pages`.

## Rollout / verification
- Do the work on a branch and open a PR.
- Before merging: switch `build_type` (an outward-facing step, so confirm with Levi first). Then merge, watch the workflow, and check that `curl` shows the new domain returns 200, `src/output.css` returns 200 and is minified, and the meta tags are present.
