# Deploy to Sandbar

GitHub Action for deploying static sites to [Sandbar](https://sandbar.cloud).

Authenticates via GitHub OIDC — no API keys needed. Configure your site to
trust your repository under **Settings > OIDC Trust** in the Sandbar console.

## Quick Start

```yaml
- uses: sandbar-cloud/sandbar-action@v1
  with:
    dir: dist
    microwave-github-actions-exchange-id: ${{ vars.SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID }}
```

Your workflow needs `permissions: id-token: write` for OIDC authentication. The
exchange ID is the Sandbar GitHub Actions Trust Exchange created in Microwave;
pass it as the action input or expose it as
`SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID` in the workflow environment.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `dir` | No | — | Build output directory. Overrides the value in `.sandbar/config.toml`. |
| `message` | No | — | Deploy message |
| `env` | No | — | Named environment from `[env.<name>]` in `.sandbar/config.toml`. Exports `SANDBAR_ENV` so the CLI picks the right override set when running `[build] command` |
| `branch` | No | auto | Branch identifier for `sandbar deploy --branch`. On `pull_request` events the default is `pr-<number>` so previews don't overwrite production; on push events it stays empty (production). Set to `-` to force a production deploy from a PR. |
| `version` | No | `latest` | Sandbar CLI version to install |
| `working-directory` | No | `.` | Directory containing `.sandbar/config.toml` |
| `comment` | No | `auto` | Post a sticky PR comment with the preview URL. `auto` posts on `pull_request` events; `true` forces; `false` disables. |
| `comment-marker` | No | `<!-- sandbar-preview -->` | Hidden marker used to find and update the same comment on subsequent runs. Change if you deploy more than once per PR (e.g., staging + preview). |
| `comment-body` | No | `**Sandbar preview** deployed: {url}` | Markdown template. `{url}` is replaced with the deploy URL; the marker is appended automatically. |
| `github-token` | No | `${{ github.token }}` | Token used to post the PR comment. Defaults to the caller's `github.token` — usually no need to set it. The job still needs `pull-requests: write`. |
| `microwave-github-actions-exchange-id` | No | — | Microwave Trust Exchange ID used to redeem the GitHub Actions OIDC token. If omitted, set `SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID` in the workflow environment. |
| `microwave-auth-url` | No | CLI default | Microwave auth endpoint override. |
| `microwave-api-url` | No | CLI default | Microwave API endpoint override. |

## Outputs

| Output | Description |
|--------|-------------|
| `url` | Deployed site URL |
| `deploy-id` | Deploy ID |

## Example Workflows

### Production Deploy

```yaml
name: Deploy to Sandbar
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - uses: sandbar-cloud/sandbar-action@v1
        with:
          dir: dist
          microwave-github-actions-exchange-id: ${{ vars.SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID }}
```

### Production Deploy (CLI runs the build)

With `[build]` and `[env.production]` configured in `.sandbar/config.toml`,
the action runs the build itself with the right env vars injected:

```yaml
name: Deploy to Sandbar
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci

      - uses: sandbar-cloud/sandbar-action@v1
        with:
          env: production
          microwave-github-actions-exchange-id: ${{ vars.SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID }}
```

### PR Preview

The action posts a sticky comment with the preview URL automatically on
pull-request events. Subsequent pushes update the same comment instead
of creating new ones.

```yaml
name: Preview
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  preview:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - uses: sandbar-cloud/sandbar-action@v1
        with:
          dir: dist
          message: "PR #${{ github.event.pull_request.number }}"
          microwave-github-actions-exchange-id: ${{ vars.SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID }}
```

To disable the comment, set `comment: false`. To customise the body,
set `comment-body` — `{url}` is replaced with the deploy URL:

```yaml
      - uses: sandbar-cloud/sandbar-action@v1
        with:
          dir: dist
          microwave-github-actions-exchange-id: ${{ vars.SANDBAR_MICROWAVE_GITHUB_ACTIONS_EXCHANGE_ID }}
          comment-body: |
            ### Preview ready

            - URL: {url}
            - Commit: `${{ github.sha }}`
```

## Versioning

Pin to a major version tag:

```yaml
- uses: sandbar-cloud/sandbar-action@v1
```

To pin to an exact release, use the full tag (e.g. `@v1.2.0`) or a commit SHA.
