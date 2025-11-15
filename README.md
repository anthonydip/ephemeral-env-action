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
- **Docker images** built and pushed to a registry
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

### Complete Workflow with Image Building (Simple Build)

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

      # Build Docker images with PR-specific tags
      - name: Build and push images
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

          docker build -t mycompany/frontend:pr-${{ github.event.pull_request.number }} ./frontend
          docker push mycompany/frontend:pr-${{ github.event.pull_request.number }}

          docker build -t mycompany/backend:pr-${{ github.event.pull_request.number }} ./ backend
          docker push mycompany/backend:pr-${{ github.event.pull_request.number }}

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

### Complete Workflow with Image Building (Optimized Build)

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

**Template Variables:**

- `{{PR_NUMBER}}` - Automatically replaced with the PR number (e.g., `123`)

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

### Getting Your Kubeconfig

```bash
# For K3s
sudo cat /etc/rancher/k3s/k3s.yaml

# For other clusters
cat ~/.kube/config

# Optional: Base64 encode for GitHub Secrets
cat ~/.kube/config | base64
```

### Getting Your Ingress Host

The public IP or domain where your Traefik ingress controller is accessible:

```bash
# AWS EC2
curl http://169.254.169.254/latest/meta-data/public-ipv4

# Check ingress controller service
kubectl get svc -n kube-system traefik
```

## How It Works

```
PR Event → GitHub Action → Build Images → Deploy to K8s → Post Preview URL
                                                              ↓
                                                http://host/pr-123/
```

1. **PR opened/updated** → Triggers workflow
2. **(Optional)** Build Docker images with PR-specific tags
3. **Action deploys** images to isolated Kubernetes namespace (`pr-<number>`)
4. **Traefik routes** traffic to `http://<ingress-host>/pr-<number>/`
5. **GitHub comment** posted with preview URLs
6. **PR closed** → Namespace and all resources deleted

## Example Output

When a PR is created, a comment is automatically posted:

```
🚀 Preview Environment Ready!

Frontend: http://54.123.45.67/pr-123/
Backend: http://54.123.45.67/pr-123/api

The environment will be automatically deleted when this PR is closed.
```

## Troubleshooting

### Action fails with "kubeconfig not found"

- Verify `KUBECONFIG` secret is added to repository settings
- Check the kubeconfig is valid YAML or base64 encoded

### Action fails with "Config file not found"

- Ensure `.ephemeral-config.yaml` exists in your repository root
- Check the `config-path` input if using a custom location
- Make sure you have `actions/checkout@v4` before the action

### Preview URLs return 404

- Verify Traefik is installed: `kubectl get pods -n kube-system | grep traefik`
- Check `ingress-host` matches your ingress controller's public IP/domain
- Verify ingress resources exist: `kubectl get ingress -n pr-<number>`

### GitHub comment not posted

- Verify `GITHUB_TOKEN` has write permissions
- Check action logs for GitHub API errors
- Use `skip-github: true` to bypass comment integration

### Namespace won't delete

- Finalizers might be blocking deletion
- Check for stuck resources: `kubectl get all -n pr-<number>`
- Manual cleanup: `kubectl delete namespace pr-<number> --force --grace-period=0`
