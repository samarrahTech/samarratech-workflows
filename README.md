# samarratech-workflows

Reusable GitHub Actions workflows shared by all Samarratech repos (TrackExpnz, GreenCI, samarratech.com). This repo is public because public repos cannot call reusable workflows from private ones; it contains CI logic only — secrets always live in the caller repos.

## Workflows

| Workflow | Purpose |
|---|---|
| `node-ci.yml` | Typecheck / lint / test / build. Node version read from the caller's `package.json` `engines.node` — pin engines per repo, never here. |
| `e2e-playwright.yml` | Install browsers, start the app, wait, run the Playwright suite. Optional `TEST_USER_EMAIL` / `TEST_USER_PASSWORD` secrets for authed suites. |
| `deploy-cf-pages.yml` | Checks + build + `wrangler pages deploy`. |
| `deploy-cf-worker.yml` | Checks + `wrangler deploy`; optional Slack-webhook secret sync. |

All deploy workflows expect `secrets.CLOUDFLARE_API_TOKEN` (use `secrets: inherit`) and `vars.CLOUDFLARE_ACCOUNT_ID` in the caller repo.

## Calling

See `templates/caller-examples/`. Minimal CI caller:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
jobs:
  ci:
    uses: samarrahTech/samarratech-workflows/.github/workflows/node-ci.yml@v1
```

Also copy `templates/dependabot.yml` into each repo — Dependabot has no central config.

## Versioning

Callers pin `@v1`. Non-breaking changes: commit to `main`, test on the lowest-stakes caller (greenci-worker) via `@main`, then move the `v1` tag. Breaking changes: new `v2` tag, migrate callers deliberately.

```sh
git tag -f v1 && git push origin v1 --force
```
