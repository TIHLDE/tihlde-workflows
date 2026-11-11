# TIHLDE Reusable Workflows

Shared [reusable GitHub Actions workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows) used across the **TIHLDE** organisation.

| Workflow | Purpose |
|---|---|
| `_ci_ghcr.yml` | Build a Docker image and push it to GitHub Container Registry (GHCR) |
| `_notify_deploy.yml` | Tell the [Deploy Receiver](https://github.com/TIHLDE/Deploy-receiver) to deploy an image, and wait for the result |

---

## Quick start

### 1. Build & push a Docker image

```yaml
# .github/workflows/ci_prod.yml
name: CI — Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: TIHLDE/tihlde-workflows/.github/workflows/_ci_ghcr.yml@v1
    with:
      tag-prefix: ""            # empty → tags "latest" and "<sha>"
      push: ${{ github.event_name != 'pull_request' }}

  deploy:
    needs: build
    if: github.event_name != 'pull_request'
    uses: TIHLDE/tihlde-workflows/.github/workflows/_notify_deploy.yml@main
    secrets:
      DEPLOY_RECEIVER_TOKEN: ${{ secrets.DEPLOY_RECEIVER_TOKEN }}
```

### 2. Dev / staging variant

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
    uses: TIHLDE/tihlde-workflows/.github/workflows/_ci_ghcr.yml@v1
    with:
      tag-prefix: dev           # → tags "dev" and "dev-<sha>"
      push: ${{ github.event_name != 'pull_request' }}

  deploy:
    needs: build
    if: github.event_name != 'pull_request'
    uses: TIHLDE/tihlde-workflows/.github/workflows/_notify_deploy.yml@main
    with:
      tag: dev
    secrets:
      DEPLOY_RECEIVER_TOKEN: ${{ secrets.DEPLOY_RECEIVER_TOKEN }}
```

---

## Workflow reference

### `_ci_ghcr.yml` — Build & push

Builds a Docker image with [docker/build-push-action](https://github.com/docker/build-push-action) and pushes it to `ghcr.io/<owner>/<repo>`.

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `context` | `string` | `"."` | Docker build context path |
| `dockerfile` | `string` | `"./Dockerfile"` | Path to the Dockerfile |
| `tag-prefix` | `string` | `""` | Tag prefix — e.g. `dev` produces `dev` + `dev-<sha>`; empty produces `latest` + `<sha>` |
| `build-args` | `string` | `""` | Newline-separated Docker build arguments |
| `push` | `boolean` | `true` | Whether to push the image to the registry |
| `platforms` | `string` | `""` | Target platforms (e.g. `linux/amd64,linux/arm64`). Defaults to the runner platform |
| `cache-from` | `string` | `""` | Build cache source (default: `type=gha`) |
| `cache-to` | `string` | `""` | Build cache destination (default: `type=gha,mode=max`) |
| `labels` | `string` | `""` | Custom OCI labels |

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `secrets` | No | Docker build secrets to pass through |

> **Note:** `GITHUB_TOKEN` is used automatically for GHCR authentication — no extra secret needed for that.

#### Outputs

| Output | Description |
|---|---|
| `digest` | SHA256 digest of the pushed image |

#### Permissions required by the caller

```yaml
permissions:
  contents: read
  packages: write
```

---

### `_notify_deploy.yml` — Notify Deploy Receiver

Sends an HTTP POST to the [Deploy Receiver](https://github.com/TIHLDE/Deploy-receiver), then checks
the deploy's status until it finishes. The job goes red if the deploy fails, and stops waiting
after 20 minutes.

By default it deploys the short commit SHA, which `_ci_ghcr.yml` pushes, so the job must run after
a build in the same run. Rerunning an old workflow therefore deploys that old commit.

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `tag` | `string` | *short commit SHA* | Image tag to deploy. It must exist in GHCR, or the deploy fails without touching the running app. Use `tag: latest` to deploy without building. |
| `image_name` | `string` | *repo name* | Image name, when it differs from the repo name. Must match the build job. |
| `deploy_url` | `string` | `"https://deploy-king.tihlde.org/deploy"` | Full URL of the receiver's deploy endpoint, ending in `/deploy`. |

Red results that need a closer look:

| Message | Meaning |
|---|---|
| `Deploy failed in step <step>` | The deploy failed. The receiver's log on the server has the details. |
| `Receiver no longer knows this deploy` | The receiver restarted during the deploy, so the result is unknown. The deploy may have succeeded. |
| `No final deploy status after 20 minutes` | The deploy may still be running. Check the server. |

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `DEPLOY_RECEIVER_TOKEN` | **Yes** | Shared token for authenticating with the Deploy Receiver |

#### Permissions

No special permissions needed beyond the default `contents: read`.

---

## Versioning policy

This repo follows **Semantic Versioning** (SemVer):

| Tag | Meaning |
|---|---|
| `v1.0.0`, `v1.1.0`, … | Immutable point releases |
| `v1` | **Moving** major tag — always points to the latest `v1.x.x` |

### Recommendations for callers

| Approach | Example | Trade-off |
|---|---|---|
| **Pin to major tag** (recommended) | `@v1` | Automatic minor/patch updates, manual major bumps |
| **Pin to exact version** | `@v1.2.0` | Full control, must update manually |
| **Pin to commit SHA** (maximum security) | `@<full-sha>` | Immutable, most secure for supply-chain protection |

> For most TIHLDE repos, pinning to `@v1` is the right balance of convenience and safety.

---

## Safe event gating for public repos

Since these workflows are used in **public** repositories, follow these rules:

| Event | Build image | Push to GHCR | Notify deploy |
|---|---|---|---|
| `pull_request` | ✅ Yes (CI check) | ❌ No | ❌ No |
| `push` (to `main`/`dev`) | ✅ Yes | ✅ Yes | ✅ Yes |
| `workflow_dispatch` | ✅ Yes | ✅ Yes | ✅ Yes |

**Why?** Pull requests from forks run with read-only `GITHUB_TOKEN` and must not push images or trigger deploys. The examples above already include the correct guards:

```yaml
push: ${{ github.event_name != 'pull_request' }}
```

```yaml
if: github.event_name != 'pull_request'
```

---

## How this connects to Deploy Receiver

```
GitHub Actions                          Server
┌──────────────────┐             ┌──────────────────────┐
│ _ci_ghcr.yml     │             │                      │
│  Build & push    │             │  Deploy Receiver     │
│  Docker image    │             │  (Node.js service)   │
└────────┬─────────┘             │                      │
         │                       │  POST /deploy        │
         ▼                       │  ├─ Validate token   │
┌──────────────────┐             │  ├─ Check apps.json  │
│ _notify_deploy   │──HTTP POST──▶  └─ docker compose up│
│  .yml            │             │                      │
└──────────────────┘             └──────────────────────┘
```

1. `_ci_ghcr.yml` builds and pushes the Docker image to GHCR.
2. `_notify_deploy.yml` sends the receiver the repo, image and tag.
3. The receiver finds the app in `apps.json`, deploys it with `docker compose`, and rolls back to the previous image if the deploy fails after the pull.
4. `_notify_deploy.yml` checks `GET /deploy/<deliveryId>` until the deploy has finished.

**Deploy Receiver repo:** <https://github.com/TIHLDE/Deploy-receiver>

---

## Repository structure

```
.github/
  dependabot.yml              # Keeps Actions pinned versions up to date
  workflows/
    _ci_ghcr.yml              # Reusable: build & push Docker image
    _notify_deploy.yml        # Reusable: notify deploy receiver
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Security

See [SECURITY.md](SECURITY.md) for our security policy and how to report vulnerabilities.

## License

[MIT](LICENSE)
