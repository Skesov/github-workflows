# github-workflows

Shared reusable GitHub Actions workflows for projects under [Skesov](https://github.com/Skesov).

## Workflows

| Workflow      | Type         | Purpose                                          |
| ------------- | ------------ | ------------------------------------------------ |
| `ci.yml`      | Orchestrator | Lint + test (for pull requests)                  |
| `cd.yml`      | Orchestrator | Lint + test → tag → build → publish → deploy     |
| `_lint.yml`   | Primitive    | Python lint with ruff via uv                     |
| `_test.yml`   | Primitive    | Python tests with pytest via uv                  |
| `_docker.yml` | Primitive    | Multi-arch Docker build and push to ghcr.io      |
| `_tag.yml`    | Primitive    | Semver tag from conventional commits             |
| `_deploy.yml` | Primitive    | Update image tag in `Skesov/homelab` GitOps repo |

Primitives can be called independently. Orchestrators compose them into a full pipeline.

## Pipeline

**Pull request** — verification only:

```
lint
test
```

**Push to master** — full delivery:

```
lint ┐
test ┘→ tag → build → publish → deploy
```

## Add to a project

### Prerequisites

- `GITOPS_TOKEN` secret on the repository — GitHub PAT with `contents: write` on `Skesov/homelab`
- HelmRelease manifest for the project in `Skesov/homelab`

```bash
gh secret set GITOPS_TOKEN --repo Skesov/my-project
```

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
    branches: [master]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: Skesov/github-workflows/.github/workflows/ci.yml@master
    with:
      ruff-paths: "src/"
      pytest-paths: "tests/"
```

### `.github/workflows/cd.yml`

```yaml
name: CD

on:
  push:
    branches: [master]

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  cd:
    uses: Skesov/github-workflows/.github/workflows/cd.yml@master
    with:
      image-name: ${{ github.repository }}
      ruff-paths: "src/"
      pytest-paths: "tests/"
      manifest-path: "flux/apps/my-project/helmrelease.yaml"
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
      gitops-token: ${{ secrets.GITOPS_TOKEN }}
    permissions:
      contents: write
      packages: write
```

## Inputs reference

### `ci.yml`

| Input            | Type   | Default  | Description            |
| ---------------- | ------ | -------- | ---------------------- |
| `python-version` | string | `"3.12"` | Python version         |
| `ruff-paths`     | string | `"."`    | Paths to lint          |
| `pytest-paths`   | string | `"."`    | Paths to test          |
| `pytest-args`    | string | `""`     | Extra pytest arguments |

### `cd.yml`

| Input            | Type   | Default                     | Description                                    |
| ---------------- | ------ | --------------------------- | ---------------------------------------------- |
| `image-name`     | string | required                    | Docker image, e.g. `owner/repo`                |
| `python-version` | string | `"3.12"`                    | Python version                                 |
| `ruff-paths`     | string | `"."`                       | Paths to lint                                  |
| `pytest-paths`   | string | `"."`                       | Paths to test                                  |
| `pytest-args`    | string | `""`                        | Extra pytest arguments                         |
| `docker-context` | string | `"."`                       | Docker build context                           |
| `platforms`      | string | `"linux/amd64,linux/arm64"` | Target platforms                               |
| `default-bump`   | string | `"patch"`                   | Semver bump when no conventional commit prefix |
| `homelab-repo`   | string | `"Skesov/homelab"`          | GitOps repo to update                          |
| `manifest-path`  | string | required                    | Path to HelmRelease in homelab repo            |
| `image-tag-path` | string | `".spec.values.image.tag"`  | yq path to image tag field                     |
| `environment`    | string | `"production"`              | GitHub Environment name                        |

### Secrets

| Secret           | Workflow | Description                                |
| ---------------- | -------- | ------------------------------------------ |
| `registry-token` | `cd.yml` | `GITHUB_TOKEN` — push to ghcr.io           |
| `gitops-token`   | `cd.yml` | PAT with `contents: write` on homelab repo |
