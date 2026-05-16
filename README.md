# github-workflows

Shared reusable GitHub Actions workflows for projects under [Skesov](https://github.com/Skesov).

Powered by [release-please](https://github.com/googleapis/release-please-action) for monorepo-aware versioning.

## Workflows

| Workflow       | Type         | Purpose                                          |
| -------------- | ------------ | ------------------------------------------------ |
| `ci.yml`       | Orchestrator | Lint + test (for pull requests)                  |
| `cd.yml`       | Orchestrator | Release-please → lint + test → build → deploy    |
| `_lint.yml`    | Primitive    | Python lint with ruff via uv                     |
| `_test.yml`    | Primitive    | Python tests with pytest via uv                  |
| `_docker.yml`  | Primitive    | Multi-arch Docker build and push to ghcr.io      |
| `_release.yml` | Primitive    | Wraps `googleapis/release-please-action`         |
| `_deploy.yml`  | Primitive    | Update image tag in `Skesov/homelab` GitOps repo |

Primitives can be called independently. Orchestrators compose them into a full pipeline.

## Pipeline

**Pull request** — verification only:

```text
lint
test
```

**Push to master** — release-driven:

```text
release ─┐
lint  ───┤
test  ───┤
         └→ build (matrix per released component) → deploy (matrix)
```

`release-please` only triggers `build` + `deploy` when a release PR is merged
(i.e. when at least one component gets a new tag). On other pushes the action
only opens or updates the release PR — `lint` and `test` still run, `build`
and `deploy` are skipped.

## Add to a project

### Prerequisites

- `GITOPS_TOKEN` repo secret — GitHub PAT with `contents: write` on `Skesov/homelab`
- HelmRelease manifest for the project in `Skesov/homelab`
- `release-please-config.json` and `.release-please-manifest.json` at repo root
  (see [release-please docs](https://github.com/googleapis/release-please/blob/main/docs/manifest-releaser.md))

```bash
gh secret set GITOPS_TOKEN --repo Skesov/my-project
```

### Caller workflow files

#### `.github/workflows/ci.yml`

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
    secrets: inherit
```

#### `.github/workflows/cd.yml`

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
    secrets: inherit
    permissions:
      contents: write
      packages: write
      pull-requests: write
```

### Single-image release-please config

For a single-image project, place these two files at the repo root:

**`release-please-config.json`**:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "python",
      "package-name": "my-project"
    }
  }
}
```

**`.release-please-manifest.json`**:

```json
{ ".": "0.1.0" }
```

Tags created: `v0.2.0`, `v0.3.0`, ... (no component prefix).

### Monorepo with multiple images

Example: a Python backend (`bot/`) and a Vite/React frontend (`web/`) in one repo,
each with its own Docker image and HelmRelease.

**`release-please-config.json`**:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    "bot": {
      "release-type": "python",
      "package-name": "sub-manager-bot",
      "component": "bot",
      "include-component-in-tag": true
    },
    "web": {
      "release-type": "node",
      "package-name": "sub-manager-bot-web",
      "component": "web",
      "include-component-in-tag": true
    }
  }
}
```

**`.release-please-manifest.json`**:

```json
{ "bot": "0.1.0", "web": "0.1.0" }
```

Tags created: `bot-v0.2.0`, `web-v0.1.1`, etc. — independent per component.

**`.github/workflows/cd.yml`** on the caller:

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
      ruff-paths: "bot/"
      pytest-paths: "bot/tests/"
      images: |
        - name: ${{ github.repository }}
          package-path: bot
          context: .
          dockerfile: bot/Dockerfile
          cache-scope: bot
          manifest-path: flux/apps/sub-manager-bot/bot.yaml
        - name: ${{ github.repository }}-web
          package-path: web
          context: web
          dockerfile: web/Dockerfile
          build-args: |
            VITE_API_URL=https://api.example.com
          cache-scope: web
          manifest-path: flux/apps/sub-manager-bot/web.yaml
    secrets: inherit
    permissions:
      contents: write
      packages: write
      pull-requests: write
```

`package-path` must match a key in `release-please-config.json`. When release-please
bumps only one component, only that component's image is built and deployed.

## Inputs reference

### `ci.yml`

| Input            | Type   | Default  | Description                                   |
| ---------------- | ------ | -------- | --------------------------------------------- |
| `python-version` | string | `"3.12"` | Python version                                |
| `ruff-paths`     | string | `"."`    | Paths to lint                                 |
| `pytest-paths`   | string | `"."`    | Paths to test                                 |
| `pytest-args`    | string | `""`     | Extra pytest arguments                        |
| `pytest-env`     | string | `"{}"`   | Env vars for pytest as JSON, e.g. `{"K":"V"}` |

### `cd.yml`

#### Lint and test

| Input            | Type   | Default  | Description                 |
| ---------------- | ------ | -------- | --------------------------- |
| `python-version` | string | `"3.12"` | Python version              |
| `ruff-paths`     | string | `"."`    | Paths to lint               |
| `pytest-paths`   | string | `"."`    | Paths to test               |
| `pytest-args`    | string | `""`     | Extra pytest arguments      |
| `pytest-env`     | string | `"{}"`   | Env vars for pytest as JSON |

#### Single-image mode (when `images` is empty)

| Input            | Type   | Default                        | Description                                        |
| ---------------- | ------ | ------------------------------ | -------------------------------------------------- |
| `image-name`     | string | `""` (required if no `images`) | Docker image, e.g. `owner/repo`                    |
| `docker-context` | string | `"."`                          | Docker build context                               |
| `dockerfile`     | string | `""`                           | Path to Dockerfile; empty → `<context>/Dockerfile` |
| `build-args`     | string | `""`                           | Newline `KEY=value` build args                     |
| `cache-scope`    | string | `""`                           | GHA cache scope                                    |
| `manifest-path`  | string | `""` (required if no `images`) | HelmRelease path in homelab                        |
| `tag-paths`      | string | `".spec.values.image.tag"`     | Newline-separated yq paths to update               |

#### Multi-image mode

| Input    | Type   | Default | Description                                    |
| -------- | ------ | ------- | ---------------------------------------------- |
| `images` | string | `""`    | YAML list of component image specs (see below) |

Each entry in `images` accepts: `name` (required), `package-path` (required, must
match release-please config), `context`, `dockerfile`, `build-args`, `cache-scope`,
`manifest-path` (required), `tag-paths`, `platforms`.

#### Shared

| Input          | Type   | Default                     | Description                             |
| -------------- | ------ | --------------------------- | --------------------------------------- |
| `platforms`    | string | `"linux/amd64,linux/arm64"` | Default platforms (per-image override)  |
| `homelab-repo` | string | `"Skesov/homelab"`          | GitOps repo to update                   |
| `environment`  | string | `"production"`              | GitHub Environment for protection rules |

#### Release-please

| Input                   | Type   | Default    | Description                                      |
| ----------------------- | ------ | ---------- | ------------------------------------------------ |
| `target-branch`         | string | `"master"` | Branch release-please tracks                     |
| `release-config-file`   | string | `""`       | Override path to `release-please-config.json`    |
| `release-manifest-file` | string | `""`       | Override path to `.release-please-manifest.json` |

### Secrets

| Secret           | Workflow | Description                                |
| ---------------- | -------- | ------------------------------------------ |
| `registry-token` | `cd.yml` | `GITHUB_TOKEN` — push to ghcr.io           |
| `gitops-token`   | `cd.yml` | PAT with `contents: write` on homelab repo |

All workflows use `secrets: inherit`. Any repo secret is automatically
exported as an environment variable in the pytest step via `toJSON(secrets)`.
No workflow changes needed when adding new secrets — set the secret on the
caller repo and tests pick it up.

## Migrating from the old `cd.yml` (mathieudutour)

The old `cd.yml` auto-bumped a tag on every push to master via
`mathieudutour/github-tag-action`. The new flow uses `release-please-action`
and a release-PR pattern.

For each caller repo:

1. Add `release-please-config.json` and `.release-please-manifest.json` at the
   repo root (see [Single-image](#single-image-release-please-config) or
   [Monorepo](#monorepo-with-multiple-images)).
2. Set the initial manifest version to your current tag's version (without `v`).
3. Update the caller workflow's `permissions:` block to add `pull-requests: write`.
4. Remove the `default-bump` input from the `cd.yml` call (no longer supported —
   release-please derives bump from conventional commits).
5. First push after migration opens a release PR; merge it to trigger the actual
   release. Subsequent commits update the PR until merged.

Tag format stays `v1.2.3` for single-image repos. Monorepo repos move to
`bot-v1.2.3` / `web-v0.5.1` style via `include-component-in-tag: true`.
