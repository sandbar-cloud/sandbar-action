# Deploy to Sandbar

GitHub Action for deploying static sites to [Sandbar](https://sandbar.cloud).

Authenticates via GitHub OIDC — no API keys needed. Configure your site to trust your repository under **Settings > OIDC Trust** in the Sandbar console.

## Quick Start

```yaml
- uses: sandbar-cloud/sandbar-action@v1
  with:
    dir: dist
```

Your workflow needs `permissions: id-token: write` for OIDC authentication.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `dir` | No | — | Build output directory. Overrides the value in `.sandbar/config.toml`. |
| `message` | No | — | Deploy message |
| `version` | No | `latest` | Sandbar CLI version to install |
| `working-directory` | No | `.` | Directory containing `.sandbar/config.toml` |

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
```

### PR Preview

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

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - uses: sandbar-cloud/sandbar-action@v1
        id: preview
        with:
          dir: dist
          message: "PR #${{ github.event.pull_request.number }}"

      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed: ${{ steps.preview.outputs.url }}`
            })
```

## Versioning

Pin to a major version tag:

```yaml
- uses: sandbar-cloud/sandbar-action@v1
```

To pin to an exact release, use the full tag (e.g. `@v1.2.0`) or a commit SHA.
