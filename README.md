# Levi Chiang — Resume

[![Deploy](https://github.com/levijcl/resume/actions/workflows/deploy.yml/badge.svg)](https://github.com/levijcl/resume/actions/workflows/deploy.yml)

Personal resume and CV, built as static HTML pages styled with Tailwind CSS.

## Live

- **Resume:** <https://resume.levijcl.com>
- **CV:** <https://resume.levijcl.com/cv>

## Stack

- Static HTML (`index.html`, `cv/index.html`)
- [Tailwind CSS v4](https://tailwindcss.com/) via the standalone CLI
- GitHub Pages, deployed by GitHub Actions

## Development

```bash
npm ci
npm run dev   # rebuilds src/output.css on change
```

Then open `index.html` (or `cv/index.html`) in a browser.

> `src/output.css` is generated and **not committed**. Pages render unstyled
> until you run `npm run dev` or `npm run build` once.

For a one-off minified build: `npm run build`.

## Print / PDF

The resume page (`index.html`) includes A4 print styles. Use the browser's **Print → Save as PDF**.

## Deployment

Every push to `main` runs [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml):
install, `npm run build`, stage `index.html`, `cv/`, `src/output.css` into `_site/`,
then deploy to GitHub Pages. The custom domain `resume.levijcl.com` is configured
in the repository's Pages settings.

**Rollback:** revert the offending commit (or re-run an earlier workflow run). Don't switch
Pages back to "Deploy from a branch" — `main` no longer contains `src/output.css`.

## Dependency updates

Dependabot (`.github/dependabot.yml`) checks npm packages and GitHub Actions every
Monday 06:00 (Asia/Taipei) and opens grouped PRs. Pull requests run the build (no deploy),
so a broken upgrade shows up before merge.

## License

[MIT](LICENSE)
