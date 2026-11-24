# Actions Deploy + resume.levijcl.com Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy the static resume through GitHub Actions instead of committing build output, and surface `resume.levijcl.com` in the pages, meta tags, README and repo About.

**Architecture:** On push to `main`, a single workflow builds Tailwind (`--minify`), stages the static files into `_site/` and deploys them with `actions/deploy-pages`. The Pages source switches from legacy branch builds to the workflow before the PR merges.

**Tech Stack:** Static HTML, Tailwind CSS v4 CLI, GitHub Actions, GitHub Pages, `gh` CLI.

**Spec:** `docs/superpowers/specs/2026-11-24-actions-deploy-design.md`

## Global Constraints

- No bundler and no framework. The site stays `index.html` + `cv/index.html` + `src/output.css`.
- Resume content, layout and the `@media print` styles stay unchanged.
- Node 22. Use the current major versions of `actions/checkout`, `actions/setup-node`, `actions/configure-pages`, `actions/upload-pages-artifact` and `actions/deploy-pages`.
- The canonical domain is `https://resume.levijcl.com/` (resume) and `https://resume.levijcl.com/cv/` (CV).
- Hrefs stay relative (`./cv`, `../`). Only the displayed link text changes.
- Anything outward-facing (the Pages `build_type` switch, `gh repo edit`, PR merge) needs Levi's confirmation first.
- Work happens on the branch `chore/actions-deploy`.

## Review Focus

- **`cv/index.html` loads `../src/output.css`.** The staged `_site/` must keep the `src/` path, otherwise the CV page renders unstyled. Task 1 Step 4 checks this.
- **Tailwind content scanning.** `src/input.css` `@source`s both HTML files. A minified build must still contain the classes both pages use, for example `sm:basis-1/5`. Task 1 Step 3 checks this.
- **Fresh clone without `output.css`.** Opening `index.html` locally gives unstyled pages until `npm run dev` or `npm run build` runs. The README must state this (Task 3).
- **Cutover gap.** If the PR merges while `build_type` is still `legacy`, the legacy build serves a site with no CSS. Task 4 switches the setting before merging.
- **The CNAME file is ignored by Actions deploys.** The custom domain must still answer 200 after the deploy. Task 4 Step 5 checks this.

---

### Task 1: Build scripts, deploy workflow, untrack output

**Files:**
- Modify: `package.json`
- Modify: `.gitignore`
- Create: `.github/workflows/deploy.yml`
- Delete from git: `src/output.css` (`git rm --cached`)

**Interfaces:**
- Produces: `npm run dev` runs `tailwindcss -i ./src/input.css -o ./src/output.css --watch`, and `npm run build` runs the same input and output with `--minify`. Tasks 3 and 4 refer to these names.

- [ ] **Step 1:** Create the branch: `git switch -c chore/actions-deploy`. Then commit the spec and this plan: `git add docs && git commit -m "docs: add Actions deploy design and plan"`.
- [ ] **Step 2:** Add the `dev` and `build` scripts to `package.json`, and add `src/output.css` to `.gitignore`. Then run `git rm --cached src/output.css`.
- [ ] **Step 3:** Verify the build:
  ```bash
  rm -f src/output.css && npm ci && npm run build
  test $(wc -l < src/output.css) -le 3 && grep -q 'basis-1\/5' src/output.css && echo OK
  ```
  Expected: `OK`. The output is minified (few lines) and contains a class from the pages.
- [ ] **Step 4:** Write `.github/workflows/deploy.yml`.
  - Triggers: `push` to `main` and `workflow_dispatch`.
  - Top-level `permissions`: `contents: read`, `pages: write`, `id-token: write`.
  - `concurrency`: group `pages`, with `cancel-in-progress: false`.
  - `build` job: checkout, then setup-node (`node-version: 22`, `cache: npm`), then `npm ci`, then `npm run build`.
  - Staging step:
    ```bash
    mkdir -p _site/src && cp -r index.html cv CNAME _site/ && cp src/output.css _site/src/
    ```
  - Then `configure-pages`, then `upload-pages-artifact` with `path: _site`.
  - `deploy` job: `needs: build`, `environment: { name: github-pages, url: ${{ steps.deployment.outputs.page_url }} }`, and one step `deploy-pages` with `id: deployment`.
  - Check the workflow locally by running the staging commands, then `ls _site _site/src _site/cv`. Expected: `index.html CNAME cv src`, `output.css`, `index.html`. Delete `_site` afterwards. If `actionlint` is installed, run `actionlint` and expect no output.
- [ ] **Step 5:** Commit: `git add -A package.json .gitignore .github src/output.css && git commit -m "ci: deploy to GitHub Pages via Actions"`.

### Task 2: Domain links + meta tags

**Files:**
- Modify: `index.html` (the `<head>`, plus the link text at line 38)
- Modify: `cv/index.html` (the `<head>`, plus the link text at line 24)

- [ ] **Step 1:** Change the link text in `index.html` to `https://resume.levijcl.com/cv`, and in `cv/index.html` to `https://resume.levijcl.com`.
- [ ] **Step 2:** Add the meta tags after `<title>` in each page. Each page gets its own `canonical` and `og:url`, with the values from Global Constraints.
  - `description`: one sentence drawn from the Summary, for example "Levi Chiang — Senior Software Engineer. Full-stack engineer with 6+ years building warehouse management, logistics, and operational systems with Java and Vue." Use the same text for both pages, but adjust the CV wording to say "Curriculum Vitae of …".
  - `og:title`: the page title prefixed with "Levi Chiang — ".
  - `og:description`: same as `description`.
  - `og:type`: `profile`.
- [ ] **Step 3:** Verify:
  ```bash
  ! grep -rn "github.io" index.html cv/index.html \
  && for f in index.html cv/index.html; do for t in 'name="description"' 'rel="canonical"' 'og:title' 'og:description' 'og:url' 'og:type'; do grep -q "$t" $f || echo "MISSING $t in $f"; done; done; echo DONE
  ```
  Expected: only `DONE`.
- [ ] **Step 4:** Commit: `git commit -am "feat: point pages at resume.levijcl.com and add meta tags"`.

### Task 3: README

**Files:**
- Modify: `README.md`

- [ ] **Step 1:** Rewrite `README.md` with these sections, in order:
  - `# Levi Chiang — Resume`, plus a one-line intro.
  - A deploy badge: `![Deploy](https://github.com/levijcl/resume/actions/workflows/deploy.yml/badge.svg)`.
  - **Live**: Resume `https://resume.levijcl.com`, CV `https://resume.levijcl.com/cv`.
  - **Stack**: static HTML, Tailwind CSS v4 CLI, GitHub Pages via Actions.
  - **Development**: `npm ci`, then `npm run dev`, then open `index.html` in a browser. Note that `src/output.css` is generated and not committed, so the pages are unstyled until you build.
  - **Print**: the page has A4 print styles; use the browser's "Save as PDF".
  - **Deployment**: pushing to `main` triggers `.github/workflows/deploy.yml`, which builds and deploys; the custom domain is set in the Pages settings.
  - **License**: link to `LICENSE`.
- [ ] **Step 2:** Verify: `grep -c "resume.levijcl.com" README.md` prints at least `2`, and `! grep -q "github.io/resume" README.md`.
- [ ] **Step 3:** Commit: `git commit -am "docs: expand README for new domain and Actions workflow"`.

### Task 4: Rollout (outward-facing — confirm each step with Levi)

- [ ] **Step 1:** Push the branch and open a PR: `git push -u origin chore/actions-deploy && gh pr create --fill`. The PR body ends with the Claude Code attribution line.
- [ ] **Step 2:** *(Confirm first)* Switch the Pages source: `gh api -X PUT repos/levijcl/resume/pages -f build_type=workflow`. Verify that `gh api repos/levijcl/resume/pages --jq '.build_type,.cname'` prints `workflow` and `resume.levijcl.com`. Then check `curl -sI https://resume.levijcl.com | head -1`; it should still be `HTTP/2 200`.
- [ ] **Step 3:** *(Confirm first)* Merge the PR, then watch the run: `gh run watch $(gh run list -w deploy.yml -L1 --json databaseId -q '.[0].databaseId')`. Expected: success.
- [ ] **Step 4:** *(Confirm first)* Set the repo homepage: `gh repo edit levijcl/resume --homepage https://resume.levijcl.com`.
- [ ] **Step 5:** Verify the live site:
  ```bash
  curl -sI https://resume.levijcl.com/ | head -1          # HTTP/2 200
  curl -sI https://resume.levijcl.com/cv/ | head -1       # HTTP/2 200
  curl -s https://resume.levijcl.com/src/output.css | wc -l   # ≤ 3 (minified)
  curl -s https://resume.levijcl.com/ | grep -c 'og:url'      # 1
  ```
  Cloudflare may cache the old responses. If the results look stale, purge the cache or retry after a few minutes before concluding anything.
