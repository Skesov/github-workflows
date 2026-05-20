# AGENTS.md

Shared reusable GitHub Actions workflows for projects under `Skesov/`. Workflows are called from other repositories via `uses: Skesov/github-workflows/.github/workflows/<file>.yml@master`.

## Architecture

Two layers:

**Orchestrators** (`ci.yml`, `cd.yml`) — compose primitives into full pipelines. External repos call only these.

**Primitives** (`_*.yml`) — single-purpose reusable workflows. Underscore prefix signals internal use.

`cd.yml` job graph:

```text
lint ┐
test ┴→ release → transform → build (matrix) → deploy (matrix)
```

`release` runs `googleapis/release-please-action@v5` after lint and test pass.
On a typical push it only opens or updates the release PR — `transform`/`build`/`deploy`
are skipped. When a release PR is merged, the action creates per-component tags and
GitHub Releases, and `transform` reads those outputs + caller's `images:` spec to
build the matrix that drives `build` and `deploy`.

Gating `release` on `lint`+`test` prevents tag creation against a broken master.

The `deploy` matrix uses `max-parallel: 1` to serialize commits against `Skesov/homelab`.

## Conventions

- Primitive job names differ from orchestrator job names to avoid triple-nesting (`cd / lint / lint`) in GitHub Actions UI. Mapping: `_release.yml` → `run`, `_docker.yml` → `push`, `_deploy.yml` → `run`.
- Pin all `uses:` references to a version tag. Verify the latest with:

  ```bash
  gh release view --repo owner/action --json tagName -q .tagName
  ```

- `homelab-repo` defaults to `Skesov/homelab`. `manifest-path` is required per component (either at top level or inside each `images[]` entry).
- Multi-image callers set `images:` (YAML list) and link each entry to a release-please package via `package-path`.

## Caller requirements

Every consumer of `cd.yml` must commit:

- `release-please-config.json` — package definitions
- `.release-please-manifest.json` — current versions

And grant `permissions: pull-requests: write` (in addition to the existing `contents: write` and `packages: write`) on the calling job.

## Validation

Run both before pushing any workflow change:

```bash
# YAML syntax and style
docker run --rm -v "$(pwd):/repo" cytopia/yamllint:latest -c /repo/.yamllint.yml /repo/.github/workflows/

# GitHub Actions semantics
docker run --rm -v "$(pwd):/repo" --workdir /repo rhysd/actionlint:latest -color
```

CI runs both automatically on push and pull request via `validate.yml`.
