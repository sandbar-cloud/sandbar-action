# Deploy to Sandbar

GitHub Action for deploying static sites to [Sandbar](https://sandbar.cloud).

Wraps the [Sandbar CLI](https://sandbar.cloud/docs/getting-started/installation) — installs the binary and runs `deploy` or `preview` against your project.

## Quick Start

```yaml
- uses: mataki-dev/sandbar-action@v1
  with:
    api-key: ${{ secrets.SANDBAR_API_KEY }}
    dir: dist
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | Yes* | — | Sandbar API key. Not required when authenticating via OIDC. |
| `command` | No | `deploy` | Command to run: `deploy` or `preview` |
| `dir` | No | — | Build output directory. Overrides the value in `.sandbar/config.toml`. |
| `version` | No | `latest` | Sandbar CLI version to install |
| `message` | No | — | Deploy message (deploy only) |
| `label` | No | — | Preview label used to identify and update the preview URL (preview only) |
| `no-activate` | No | `false` | Upload the build without making it live (deploy only) |
| `working-directory` | No | `.` | Directory containing `.sandbar/config.toml` |

## Outputs

| Output | Description |
|--------|-------------|
| `url` | Deployed site URL (production or preview) |
| `deploy-id` | Deploy ID |

## Example Workflows

### Production Deploy

Deploys to production on every push to `main`.

```yaml
name: Deploy to Sandbar
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - uses: mataki-dev/sandbar-action@v1
        with:
          api-key: ${{ secrets.SANDBAR_API_KEY }}
          dir: dist
```

### PR Preview

Creates a preview deploy for each pull request and posts the URL as a comment.

```yaml
name: Preview
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - uses: mataki-dev/sandbar-action@v1
        id: preview
        with:
          api-key: ${{ secrets.SANDBAR_API_KEY }}
          command: preview
          dir: dist
          label: pr-${{ github.event.pull_request.number }}

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

### Deploy with OIDC (No API Key)

Sandbar supports GitHub's OpenID Connect (OIDC) token exchange, so you can authenticate workflows without storing a long-lived API key as a secret. Add `permissions: id-token: write` and run `sandbar login` before the action — the CLI detects the OIDC environment automatically and exchanges the token with Sandbar.

Before using this flow, configure your Sandbar organization to trust your repository under **Settings > OIDC Trust**.

```yaml
name: Deploy to Sandbar (OIDC)
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # Required for OIDC token exchange

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci && npm run build

      - name: Install Sandbar CLI
        run: curl -sSL https://sandbar.cloud/install.sh | sh

      - name: Authenticate via OIDC
        run: sandbar login

      - uses: mataki-dev/sandbar-action@v1
        with:
          dir: dist
          # api-key is omitted — OIDC session written by `sandbar login` is used instead
```

## Versioning

Pin to a major version tag to receive non-breaking updates automatically:

```yaml
- uses: mataki-dev/sandbar-action@v1
```

To pin to an exact release, use the full tag (e.g. `@v1.2.0`) or a commit SHA.
