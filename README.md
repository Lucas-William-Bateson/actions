# Reusable GitHub Actions for Dokku

Composite actions for deploying to [Dokku](https://dokku.com/) via `git push`.

## Actions

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

## Example Workflows

### Production deploy (push to main)

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: Lucas-William-Bateson/actions/dokku-deploy@main
        with:
          app: my-app
          dockerfile: docker/Dockerfile
```

### PR preview apps

```yaml
name: Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: Lucas-William-Bateson/actions/dokku-deploy@main
        id: deploy
        with:
          app: my-app-pr-${{ github.event.number }}
          dockerfile: docker/Dockerfile
          domains: my-app-pr-${{ github.event.number }}.l3s.me
          ports: http:80:3000

      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 Preview: ${{ steps.deploy.outputs.url }}`
            })

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: self-hosted
    steps:
      - uses: Lucas-William-Bateson/actions/dokku-cleanup@main
        with:
          app: my-app-pr-${{ github.event.number }}
```

---

## Prerequisites

### Self-hosted runner

A GitHub Actions runner must be running on the Dokku host (or a machine with SSH access to it).

```bash
# On the Mac Mini:
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-osx-arm64-2.322.0.tar.gz
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/Lucas-William-Bateson --token <REGISTRATION_TOKEN>
./svc.sh install
./svc.sh start
```

Get the registration token from: **GitHub Org Settings → Actions → Runners → New self-hosted runner**

### Repository access

If this repo is private, allow other repos to use its actions:
**Repo Settings → Actions → General → Access → Accessible from repositories in the organization**
