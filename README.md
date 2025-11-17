# Ephemeral Environment Action

GitHub Action for creating and deleting ephemeral Kubernetes preview environments for pull requests.

> **Powered by:** [ephemeral-env-platform](https://github.com/anthonydip/ephemeral-env-platform)

## Features

- Isolated Kubernetes environments for each PR
- Automatic GitHub PR comments with preview URLs
- Auto-cleanup when PR closes
- Multi-service support per environment
- Traefik-based routing with path prefixes
- Environment variable injection
- Template variable support (`{{PR_NUMBER}}`)
- Works with any cloud provider (AWS, GCP, Azure, DigitalOcean, etc.)

## Prerequisites

- **Kubernetes cluster** (K3s, GKE, EKS, AKS, etc.)
- **Traefik ingress controller** installed
- **Docker images** built and pushed to a registry (depending on workflow setup)
- **`.ephemeral-config.yaml`** in your repository

## Quick Start

### Basic Workflow

```yaml
name: Preview environments

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  manage-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Manage ephemeral environment
        uses: anthonydip/ephemeral-env-action@v1
        with:
          action: ${{ github.event.action == 'closed' && 'delete' || 'create' }}
          pr-number: ${{ github.event.pull_request.number }}
          kubeconfig: ${{ secrets.KUBECONFIG }}
          ingress-host: ${{ secrets.INGRESS_HOST }}
```

### Complete Workflow with Image Building

```yaml
name: Preview Environments

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  build-and-deploy:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push frontend image
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: mycompany/frontend:pr-${{ github.event.pull_request.number }}
          cache-from: type=registry,ref=mycompany/frontend:buildcache
          cache-to: type=registry,ref=mycompany/frontend:buildcache,mode=max

      - name: Build and push backend image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: mycompany/backend:pr-${{ github.event.pull_request.number }}
          cache-from: type=registry,ref=mycompany/backend:buildcache
          cache-to: type=registry,ref=mycompany/backend:buildcache,mode=max

      # Deploy preview environment
      - name: Create preview environment
        uses: anthonydip/ephemeral-env-action@v1
        with:
          action: create
          pr-number: ${{ github.event.pull_request.number }}
          kubeconfig: ${{ secrets.KUBECONFIG }}
          ingress-host: ${{ secrets.INGRESS_HOST }}

  cleanup:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Delete preview environment
        uses: anthonydip/ephemeral-env-action@v1
        with:
          action: delete
          pr-number: ${{ github.event.pull_request.number }}
          kubeconfig: ${{ secrets.KUBECONFIG }}
          ingress-host: ${{ secrets.INGRESS_HOST }}
```

## Configuration

### Create `.ephemeral-config.yaml`

Create this file in your repository root:

```yaml
services:
  - name: frontend
    image: mycompany/frontend:pr-{{PR_NUMBER}}
    port: 80
    ingress:
      enabled: true
      path: "/"

  - name: backend
    image: mycompany/backend:pr-{{PR_NUMBER}}
    port: 3000
    ingress:
      enabled: true
      path: "/api"
    env:
      NODE_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
```

### Template Variables

The action automatically processes your config file and replaces template variables:

- `{{PR_NUMBER}}` - Replaced with the actual PR number (e.g., `123` becomes `mycompany/frontend:pr-123`)

**Why use template variables?**

- One config file works for all PRs (no manual updates)
- Your workflow builds images with PR-specific tags (see build examples above)
- The action automatically matches your built images to the config

### Input Reference

| Input          | Description                               | Required | Default                  |
| -------------- | ----------------------------------------- | -------- | ------------------------ |
| `action`       | Action to perform (`create` or `delete`)  | **Yes**  | -                        |
| `pr-number`    | Pull request number                       | **Yes**  | -                        |
| `kubeconfig`   | Kubernetes config for cluster access      | **Yes**  | -                        |
| `ingress-host` | Public IP or domain of ingress controller | **Yes**  | -                        |
| `github-token` | GitHub token for PR comments              | No       | `${{ github.token }}`    |
| `config-path`  | Path to config file                       | No       | `.ephemeral-config.yaml` |
| `template-dir` | Path to template directory                | No       | `automation/templates/`  |
| `log-level`    | Logging level                             | No       | `INFO`                   |
| `skip-github`  | Skip GitHub PR comment integration        | No       | `false`                  |

## Required GitHub Secrets

Add these secrets to your repository settings (Settings -> Secrets and variables -> Actions):

| Secret            | Description                                    | Example                                 |
| ----------------- | ---------------------------------------------- | --------------------------------------- |
| `KUBECONFIG`      | Your Kubernetes cluster configuration          | (base64 or raw YAML)                    |
| `INGRESS_HOST`    | Public IP or domain of ingress controller      | `54.123.45.67` or `preview.company.com` |
| `DOCKER_USERNAME` | (If building images) Docker Hub username       | -                                       |
| `DOCKER_PASSWORD` | (If building images) Docker Hub password/token | -                                       |

## Example Output

When a PR is created, a comment is automatically posted:

```
🚀 Preview Environment Ready!

Frontend: http://54.123.45.67/pr-123/
Backend: http://54.123.45.67/pr-123/api

The environment will be automatically deleted when this PR is closed.
```

## Security Considerations

This action requires access to sensitive secrets (kubeconfig, ingress host). To prevent unauthorized access:

**For public repositories**, restrict workflows to same-repo PRs only:

```yaml
jobs:
  manage-preview:
  if: github.event.pull_request.head.repo.full_name == github.repository
  runs-on: ubuntu-latest
  # ... rest of job
```

This prevents PRs from forks from attempting to access your Kubernetes cluster.

For private repositories, this is not a concern as only collaborators can create PRs.
