# Reusable GitHub Actions for Dokku

Composite actions and reusable workflows for deploying to [Dokku](https://dokku.com/).

## Reusable Workflow

### `dokku-deploy.yml`

The recommended way to deploy. All deploy logic lives here — each repo just calls it with app-specific inputs.

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened, closed]
  schedule:
    - cron: '0 6 * * 1' # Weekly rebuild

jobs:
  deploy:
    uses: Lucas-William-Bateson/actions/.github/workflows/dokku-deploy.yml@main
    with:
      app: my-app
      dockerfile: docker/Dockerfile
      port: '3000'
    secrets: inherit
```

That's it. The workflow handles:
- **Push to main** → deploy to production
- **Schedule** → redeploy (rebuild with fresh dependencies)
- **PR opened/updated** → deploy preview at `my-app-pr-42.l3s.me`
- **PR closed** → destroy preview app

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app` | Yes | — | Dokku app name |
| `dockerfile` | No | — | Dockerfile path relative to repo root |
| `port` | No | — | App container port (for preview port mapping) |
| `ssh-host` | No | `100.94.15.7` | Dokku SSH host (Tailscale IP) |
| `ssh-port` | No | `3022` | Dokku SSH port |
| `preview-domain-suffix` | No | `l3s.me` | Domain suffix for PR previews |

#### Required Secrets (org-level)

| Secret | Description |
|--------|-------------|
| `SSH_PRIVATE_KEY` | SSH private key authorized with Dokku |
| `TAILSCALE_OAUTH_CLIENT_ID` | Tailscale OAuth client ID for runner networking |
| `TAILSCALE_OAUTH_SECRET` | Tailscale OAuth secret |

---

## Composite Actions

Lower-level building blocks used by the reusable workflow. You can also use these directly if you need custom logic.

### `dokku-deploy`

Deploy an application to Dokku. Creates the app if it doesn't exist, configures the builder, and pushes code.

```yaml
- uses: Lucas-William-Bateson/actions/dokku-deploy@main
  with:
    app: my-app
    dockerfile: docker/Dockerfile  # optional
```

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app` | Yes | — | Dokku app name |
| `ssh-host` | No | `localhost` | Dokku SSH host |
| `ssh-port` | No | `3022` | Dokku SSH port |
| `ssh-private-key` | No | — | SSH private key (uses runner default key if omitted) |
| `dockerfile` | No | — | Dockerfile path relative to repo root |
| `domains` | No | — | Space-separated domains to assign |
| `ports` | No | — | Port mapping (e.g., `http:80:3000`) |
| `branch` | No | `main` | Target branch on Dokku remote |

#### Outputs

| Output | Description |
|--------|-------------|
| `url` | Primary URL of the deployed app |

---

### `dokku-cleanup`

Destroy a Dokku app. Used for tearing down PR preview environments.

```yaml
- uses: Lucas-William-Bateson/actions/dokku-cleanup@main
  with:
    app: my-app-pr-42
```

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `app` | Yes | — | Dokku app name to destroy |
| `ssh-host` | No | `localhost` | Dokku SSH host |
| `ssh-port` | No | `3022` | Dokku SSH port |
| `ssh-private-key` | No | — | SSH private key |

---

## Setup

### 1. Tailscale OAuth client

Runners connect to the Mac Mini via Tailscale. Create an OAuth client:

1. Go to [Tailscale Admin → Settings → OAuth clients](https://login.tailscale.com/admin/settings/oauth)
2. Create a new client with `devices:write` scope
3. Add the client ID and secret as org secrets (see below)
4. Create an ACL tag `tag:ci` and allow it access to the Dokku host

### 2. GitHub org secrets

Set these at **GitHub Org Settings → Secrets and variables → Actions**:

| Secret | Value |
|--------|-------|
| `SSH_PRIVATE_KEY` | SSH private key authorized with Dokku (`ssh dokku-target`) |
| `TAILSCALE_OAUTH_CLIENT_ID` | From step 1 |
| `TAILSCALE_OAUTH_SECRET` | From step 1 |

### 3. Repository access

If this repo is private, allow other repos to use its actions:
**Repo Settings → Actions → General → Access → Accessible from repositories in the organization**
