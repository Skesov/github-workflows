# github-workflows

Shared reusable GitHub Actions workflows for projects under [Skesov](https://github.com/Skesov).

Powered by [release-please](https://github.com/googleapis/release-please-action) for monorepo-aware versioning.

## Workflows

| Workflow           | Type         | Purpose                                                |
| ------------------ | ------------ | ------------------------------------------------------ |
| `ci.yml`           | Orchestrator | Python lint + test (for pull requests)                 |
| `cd.yml`           | Orchestrator | Lint + test → release-please → build → deploy          |
| `_lint-python.yml` | Primitive    | ruff check + ruff format via uv                        |
| `_test-python.yml` | Primitive    | pytest via uv                                          |
| `_lint-node.yml`   | Primitive    | typecheck + lint via pnpm/npm/yarn                     |
| `_test-node.yml`   | Primitive    | unit tests + optional Playwright e2e                   |
| `_lint-go.yml`     | Primitive    | golangci-lint                                          |
| `_test-go.yml`     | Primitive    | `go test`                                              |
| `_docker.yml`      | Primitive    | Docker build (native arm64 runner) and push to ghcr.io |
| `_release.yml`     | Primitive    | Wraps `googleapis/release-please-action`               |
| `_deploy.yml`      | Primitive    | Update image tag in `Skesov/homelab` GitOps repo       |

Primitives can be called independently — staging builds and manual redeploys call
`_docker.yml` and `_deploy.yml` directly. Orchestrators compose them into a full pipeline.

## Pipeline

**Pull request** — verification only:

```text
lint
test
```

**Push to master** — release-driven:

```text
changes ─→ lint-python / test-python ─┐
          lint-node   / test-node    ─┤
          lint-go     / test-go      ─┴→ release → transform → build (matrix) → deploy (matrix)
```

`changes` turns the caller's `components` list into dorny/paths-filter config and
builds one lint/test matrix per stack, holding only the components whose `paths`
changed in the push. A repo that declares one stack leaves the other matrices empty
and those jobs are skipped.

`release-please` only triggers `build` + `deploy` when a release PR is merged
(i.e. when at least one component gets a new tag). On other pushes the action
only opens or updates the release PR — `lint` and `test` still run, `build`
and `deploy` are skipped.

GitHub propagates a skip to every descendant whose `if` has no status function,
which once silently swallowed the whole `transform → build → deploy` tail on a
real release. `transform`, `build` and `deploy` therefore guard on
`!cancelled() && needs.<prev>.result == 'success'` — keep that shape when editing them.

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

`ci.yml` covers Python only. For Node or Go pull-request checks, call
`_lint-node.yml` / `_test-node.yml` / `_lint-go.yml` / `_test-go.yml` directly.

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

Single Python service, one image, one HelmRelease:

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
      components: |
        - name: bot
          stack: python
          paths:
            - "src/**"
            - "tests/**"
            - "Dockerfile"
            - "pyproject.toml"
            - "uv.lock"
          package-path: "."
          setup:
            python-version: "3.12"
          lint:
            paths: "src/"
          test:
            paths: "tests/"
          image:
            name: ${{ github.repository }}
          deploy:
            manifest-path: "flux/apps/my-project/helmrelease.yaml"
            tag-paths: |
              .spec.values.image.tag
              .metadata.annotations["event.toolkit.fluxcd.io/version"]
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
      gitops-token: ${{ secrets.GITOPS_TOKEN }}
    permissions:
      contents: write
      packages: write
      pull-requests: write
```

`secrets:` must be passed explicitly. GitHub secret names cannot contain hyphens,
so `secrets: inherit` would not match the `registry-token` / `gitops-token`
parameters and the job would fail at runtime.

## Components schema

`components` is a YAML list passed as a string. One entry per releasable unit:
its sources, its checks, its image, its manifest.

| Key            | Required | Description                                                            |
| -------------- | -------- | ---------------------------------------------------------------------- |
| `name`         | yes      | Logical name. Matrix key and dorny filter id — must be unique          |
| `stack`        | yes      | `python`, `node` or `go`. Picks the lint/test primitives               |
| `paths`        | yes      | Globs for dorny/paths-filter. Lint and test run only when these change |
| `package-path` | yes      | release-please key. Must match a key in `release-please-config.json`   |
| `setup`        | no       | Runtime and workspace settings (see below)                             |
| `lint`         | no       | Lint settings (see below)                                              |
| `test`         | no       | Test settings (see below)                                              |
| `image`        | yes      | Docker build settings (see below)                                      |
| `deploy`       | yes      | GitOps settings (see below)                                            |

### `setup`

| Key                 | Stacks   | Default  | Description                               |
| ------------------- | -------- | -------- | ----------------------------------------- |
| `python-version`    | python   | `3.12`   | Python version for uv                     |
| `node-version`      | node     | `22`     | Node.js version                           |
| `go-version`        | go       | `stable` | Go version                                |
| `package-manager`   | node     | `pnpm`   | `pnpm`, `npm` or `yarn`                   |
| `install-command`   | node     | derived  | Override the install command              |
| `working-directory` | node, go | `.`      | Directory holding `package.json`/`go.mod` |

### `lint`

| Key        | Stacks | Default                       | Description                        |
| ---------- | ------ | ----------------------------- | ---------------------------------- |
| `paths`    | python | `.`                           | Paths passed to ruff               |
| `commands` | node   | `<pm> typecheck`, `<pm> lint` | Newline-separated commands         |
| `version`  | go     | `latest`                      | golangci-lint version tag          |
| `args`     | go     | `""`                          | Extra args for `golangci-lint run` |

### `test`

| Key                      | Stacks     | Default               | Description                                   |
| ------------------------ | ---------- | --------------------- | --------------------------------------------- |
| `paths`                  | python, go | `.` / `./...`         | Test paths                                    |
| `args`                   | python, go | `""` / `-race -cover` | Extra args                                    |
| `commands`               | node       | `<pm> test`           | Newline-separated test commands               |
| `env`                    | all        | `{}`                  | Env vars as a YAML map, exported before tests |
| `e2e-command`            | node       | `""`                  | Runs after `commands`. Empty skips e2e        |
| `playwright`             | node       | `false`               | Install and cache Playwright Chromium         |
| `playwright-report-path` | node       | `playwright-report/`  | Artifact uploaded when e2e fails              |

### `image`

| Key           | Required | Default                | Description                                               |
| ------------- | -------- | ---------------------- | --------------------------------------------------------- |
| `name`        | yes      | —                      | Image name, e.g. `owner/repo`                             |
| `context`     | no       | `.`                    | Docker build context                                      |
| `dockerfile`  | no       | `<context>/Dockerfile` | Path to Dockerfile, relative to repo root                 |
| `build-args`  | no       | `""`                   | Newline-separated `KEY=value` build args                  |
| `cache-scope` | no       | `""`                   | GHA cache scope. Set it when a repo builds several images |
| `platforms`   | no       | `default-platforms`    | Target platforms                                          |

### `deploy`

| Key             | Required | Default                  | Description                          |
| --------------- | -------- | ------------------------ | ------------------------------------ |
| `manifest-path` | yes      | —                        | HelmRelease path in the GitOps repo  |
| `tag-paths`     | no       | `.spec.values.image.tag` | Newline-separated yq paths to update |

## Single-image release-please config

For a single-image project, place these two files at the repo root:

**`release-please-config.json`**:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "release-type": "python",
      "package-name": "my-project",
      "include-component-in-tag": false
    }
  }
}
```

**`.release-please-manifest.json`**:

```json
{ ".": "0.1.0" }
```

Tags created: `v0.2.0`, `v0.3.0`, ... (no component prefix).

`include-component-in-tag: false` is required for the plain `v0.2.0` format.
Without it release-please prefixes the tag with `package-name`, producing
`my-project-v0.2.0`.

### uv projects: keep `uv.lock` in step

release-please bumps the version in `pyproject.toml` and never touches `uv.lock`,
which pins the workspace package version too. An image built with `uv sync --locked`
then fails with `The lockfile at uv.lock needs to be updated`; `--frozen` builds
succeed but ship a lockfile one version behind. Refresh the lockfile on the release
PR before it merges — see `release-lockfile.yml` in `Skesov/sub-manager-bot`.

## Monorepo with multiple images

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
      "package-name": "sub-manager-web",
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
jobs:
  cd:
    uses: Skesov/github-workflows/.github/workflows/cd.yml@master
    with:
      target-branch: main
      components: |
        - name: bot
          stack: python
          paths:
            - "bot/**"
            - "pyproject.toml"
            - "uv.lock"
          package-path: "bot"
          lint:
            paths: "bot/"
          test:
            paths: "bot/tests/"
          image:
            name: ${{ github.repository }}
            context: "."
            dockerfile: "bot/Dockerfile"
            cache-scope: prod-bot
          deploy:
            manifest-path: "flux/apps/sub-manager-bot/helmrelease.yaml"
        - name: web
          stack: node
          paths:
            - "web/**"
          package-path: "web"
          setup:
            node-version: "22"
            package-manager: pnpm
            working-directory: "web"
          lint:
            commands: |
              pnpm typecheck
              pnpm lint
          test:
            commands: pnpm test
            e2e-command: pnpm e2e
            playwright: true
          image:
            name: ${{ github.repository_owner }}/sub-manager-web
            context: "web"
            dockerfile: "web/Dockerfile"
            build-args: |
              VITE_API_URL=https://api.example.com
            cache-scope: prod-web
          deploy:
            manifest-path: "flux/apps/sub-manager-bot/helmrelease-web.yaml"
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
      gitops-token: ${{ secrets.GITOPS_TOKEN }}
    permissions:
      contents: write
      packages: write
      pull-requests: write
```

`package-path` must match a key in `release-please-config.json`. When release-please
bumps only one component, only that component's image is built and deployed. A
release whose `package-path` matches no component fails `transform` with an explicit
error rather than deploying nothing.

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

| Input                   | Type   | Default            | Description                                      |
| ----------------------- | ------ | ------------------ | ------------------------------------------------ |
| `components`            | string | required           | YAML list of components (see schema above)       |
| `target-branch`         | string | `"master"`         | Branch release-please tracks                     |
| `release-config-file`   | string | `""`               | Override path to `release-please-config.json`    |
| `release-manifest-file` | string | `""`               | Override path to `.release-please-manifest.json` |
| `homelab-repo`          | string | `"Skesov/homelab"` | GitOps repo to update                            |
| `environment`           | string | `"production"`     | GitHub Environment for protection rules          |
| `default-platforms`     | string | `"linux/arm64"`    | Platforms when a component sets none             |

Outputs: `releases_created` (string `"true"`/`"false"`) and `released_packages`
(JSON array of `{path, tag, version}`).

### `_docker.yml`

| Input         | Type    | Default              | Description                               |
| ------------- | ------- | -------------------- | ----------------------------------------- |
| `image-name`  | string  | required             | Image name, e.g. `owner/repo`             |
| `version`     | string  | required             | Version tag, e.g. `1.2.3`                 |
| `registry`    | string  | `"ghcr.io"`          | Container registry                        |
| `platforms`   | string  | `"linux/arm64"`      | Target platforms                          |
| `runner`      | string  | `"ubuntu-24.04-arm"` | Runner label; the default builds natively |
| `context`     | string  | `"."`                | Build context                             |
| `dockerfile`  | string  | `""`                 | Empty means `<context>/Dockerfile`        |
| `build-args`  | string  | `""`                 | Newline-separated `KEY=value`             |
| `cache-scope` | string  | `""`                 | GHA cache scope                           |
| `push-latest` | boolean | `false`              | Also push `:latest`                       |

### `_deploy.yml`

| Input           | Type   | Default                    | Description                          |
| --------------- | ------ | -------------------------- | ------------------------------------ |
| `version`       | string | required                   | Version to deploy                    |
| `manifest-path` | string | required                   | HelmRelease path in the GitOps repo  |
| `homelab-repo`  | string | `"Skesov/homelab"`         | GitOps repo                          |
| `tag-paths`     | string | `".spec.values.image.tag"` | Newline-separated yq paths to update |
| `environment`   | string | `"production"`             | GitHub Environment                   |

### `_release.yml`

| Input           | Type   | Default    | Description                             |
| --------------- | ------ | ---------- | --------------------------------------- |
| `target-branch` | string | `"master"` | Branch to release from                  |
| `config-file`   | string | `""`       | Path to `release-please-config.json`    |
| `manifest-file` | string | `""`       | Path to `.release-please-manifest.json` |

### Stack primitives

`_lint-python.yml`: `python-version`, `ruff-paths`.
`_test-python.yml`: `python-version`, `pytest-paths`, `pytest-args`, `pytest-env`.
`_lint-node.yml`: `node-version`, `package-manager`, `working-directory`, `install-command`, `lint-commands`.
`_test-node.yml`: `node-version`, `package-manager`, `working-directory`, `install-command`, `test-commands`, `test-env`, `e2e-command`, `playwright`, `playwright-report-path`.
`_lint-go.yml`: `go-version`, `working-directory`, `golangci-lint-version`, `args`.
`_test-go.yml`: `go-version`, `working-directory`, `test-paths`, `test-args`, `test-env`.

`cd.yml` maps `components[*]` onto these inputs; the defaults listed in the
[components schema](#components-schema) are the ones `cd.yml` applies.

### Secrets

| Secret           | Workflow                | Description                                |
| ---------------- | ----------------------- | ------------------------------------------ |
| `registry-token` | `cd.yml`, `_docker.yml` | `GITHUB_TOKEN` — push to ghcr.io           |
| `gitops-token`   | `cd.yml`, `_deploy.yml` | PAT with `contents: write` on homelab repo |

`ci.yml` accepts `secrets: inherit` — any caller repo secret becomes an env
var in the pytest step via `toJSON(secrets)`. Useful for test fixtures that
read API keys from the environment.

`cd.yml` requires explicit `secrets:` mapping (see examples above) and forwards
`secrets: inherit` to the test primitives, so a caller secret still reaches
pytest and `go test` on master.

## Migrating a caller to the components model

`cd.yml` before the components model took flat `ruff-paths` / `pytest-paths`
inputs plus either single-image inputs (`image-name`, `manifest-path`, ...) or an
`images:` list. All of them are gone; a caller still passing them fails at input
validation before the first job starts.

| Old input        | New location                                          |
| ---------------- | ----------------------------------------------------- |
| `ruff-paths`     | `components[].lint.paths`                             |
| `pytest-paths`   | `components[].test.paths`                             |
| `pytest-args`    | `components[].test.args`                              |
| `pytest-env`     | `components[].test.env`                               |
| `python-version` | `components[].setup.python-version`                   |
| `image-name`     | `components[].image.name`                             |
| `docker-context` | `components[].image.context`                          |
| `dockerfile`     | `components[].image.dockerfile`                       |
| `build-args`     | `components[].image.build-args`                       |
| `cache-scope`    | `components[].image.cache-scope`                      |
| `manifest-path`  | `components[].deploy.manifest-path`                   |
| `tag-paths`      | `components[].deploy.tag-paths`                       |
| `platforms`      | `components[].image.platforms` or `default-platforms` |
| `images[]`       | one `components[]` entry each                         |

New required keys with no old counterpart: `name`, `stack`, `paths`. `paths` gates
lint and test per component — list every path whose change must re-run the checks,
including the caller workflow file itself.

## Migrating from the old `cd.yml` (mathieudutour)

The old `cd.yml` auto-bumped a tag on every push to master via
`mathieudutour/github-tag-action`. The new flow uses `release-please-action`
and a release-PR pattern.

For each caller repo:

1. Enable "Allow GitHub Actions to create and approve pull requests" at
   `https://github.com/<owner>/<repo>/settings/actions`, or via API:

   ```bash
   gh api -X PUT /repos/<owner>/<repo>/actions/permissions/workflow \
     -F can_approve_pull_request_reviews=true
   ```

   Release-please cannot open the release PR without this — the first run
   will fail with
   `GitHub Actions is not permitted to create or approve pull requests`.

2. Add `release-please-config.json` and `.release-please-manifest.json` at the
   repo root (see [Single-image](#single-image-release-please-config) or
   [Monorepo](#monorepo-with-multiple-images)).
3. Set the initial manifest version to your current tag's version (without `v`).
4. Update the caller workflow's `permissions:` block to add `pull-requests: write`.
5. Replace `secrets: inherit` with explicit mapping in the `cd.yml` call (see
   examples above) — `inherit` does not match hyphenated secret parameter names.
6. Remove the `default-bump` input from the `cd.yml` call (no longer supported —
   release-please derives bump from conventional commits).
7. First push after migration opens a release PR; merge it to trigger the actual
   release. Subsequent commits update the PR until merged.

Tag format stays `v1.2.3` for single-image repos (requires
`include-component-in-tag: false` in the config). Monorepo repos move to
`bot-v1.2.3` / `web-v0.5.1` style via `include-component-in-tag: true`.

The first release after migration includes the entire commit history in
`CHANGELOG.md` (release-please has no prior tag to diff against). The
`Compare: v0.4.x...v0.5.0` link in the PR header is correct; only the body
is bloated. Edit the release PR before merging if you want a shorter
changelog.
