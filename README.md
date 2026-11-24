# samarratech-workflows

Reusable GitHub Actions workflows shared by all Samarratech repos (TrackExpnz, GreenCI, samarratech.com). This repo is public because public repos cannot call reusable workflows from private ones; it contains CI logic only — secrets always live in the caller repos.

## Workflows

| Workflow | Purpose |
|---|---|
| `node-ci.yml` | Typecheck / lint / test / build. Node version read from the caller's `package.json` `engines.node` — pin engines per repo, never here. |
| `e2e-playwright.yml` | Install browsers, start the app, wait, run the Playwright suite. Optional `TEST_USER_EMAIL` / `TEST_USER_PASSWORD` secrets for authed suites. |
| `deploy-cf-pages.yml` | Checks + build + `wrangler pages deploy`. With `branch: pr-<n>` it deploys a preview instead of production; the `deployment-url` output is the URL to test. |
| `greenci.yml` | Runs [greenci-action](https://github.com/samarrahTech/greenci-action) on a pull request against a live URL or a locally started app. `channel` picks the action ref (`v1` or `main`). Needs `secrets.GREENCI_API_KEY`; skips with a notice when it is absent. `min-tests-generated` (default 0) fails the job when fewer tests were generated (or when the run was skipped for a missing key), for canary PRs whose change is known to be testable. Outputs `tests-generated`, `tests-passed`, `tests-failed`. |
| `heldfix-check.yml` | Heldfix SEO check for a pull request: scans `baseline-url` (production) and `preview-url` (the PR preview, scanned with `--preview-of`), diffs them, and keeps one PR comment up to date. Read-only and advisory: findings never fail the job. Call it from a `pull_request` workflow; fork PRs are skipped. The scan runs in a job with no repository permissions and a separate job posts the comment. Needs `secrets.HELDFIX_REPO_TOKEN` (read-only token for the private heldfix repo); skips with a notice when it is absent. Outputs `new-findings`, `resolved-findings`, `pages-added`. |
| `heldfix-live.yml` | Post-deploy Heldfix scan of a live site: scans `site-url`, diffs it with the previous scan of that site (kept in the caller's Actions cache), and posts to Slack (`SLACK_WEBHOOK_URL`) when a new Tier 0 or Tier 1 finding appears. With no baseline (first run, or the cache entry expired after 7 days) it records one and reports the current Tier 0/1 count instead; a partial scan never replaces the baseline. Call it from the deploy workflow on the default branch, after the deploy job. Needs `secrets.HELDFIX_REPO_TOKEN`; skips with a notice without it. Output `new-t01-count`. |
| `cleanup-cf-pages-preview.yml` | Deletes every Cloudflare Pages preview deployment of one `pr-<number>` branch (what `deploy-cf-pages.yml` creates for PR previews), through the Cloudflare API with paging. Only preview deployments of that exact branch are touched; the Pages project must belong to the calling repo alone. Call it on `pull_request: closed`; see `templates/caller-examples/dogfood-cleanup.yml`. Needs `secrets.CLOUDFLARE_API_TOKEN` and `vars.CLOUDFLARE_ACCOUNT_ID`. |
| `deploy-cf-worker.yml` | Checks + `wrangler deploy`; optional Slack-webhook secret sync. |

All deploy workflows expect `secrets.CLOUDFLARE_API_TOKEN` (use `secrets: inherit`) and `vars.CLOUDFLARE_ACCOUNT_ID` in the caller repo.

## Calling

See `templates/caller-examples/` (`dogfood-pr.yml` chains a PR preview deploy into GreenCI and Heldfix). Minimal CI caller:

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
