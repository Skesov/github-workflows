# AGENTS.md

Shared reusable GitHub Actions workflows for projects under `Skesov/`. Workflows are called from other repositories via `uses: Skesov/github-workflows/.github/workflows/<file>.yml@master`.

## Architecture

Two layers:

**Orchestrators** (`ci.yml`, `cd.yml`) — compose primitives into full pipelines. External repos call only these.

**Primitives** (`_*.yml`) — single-purpose reusable workflows. Underscore prefix signals internal use.

Pipeline order in `cd.yml`:

```text
lint + test (parallel) → tag → build → push → publish → deploy
```

The `deploy` step updates the image tag in `Skesov/homelab` using `yq` and pushes a commit. Flux CD picks up the change automatically.

## Conventions

- Primitive job names differ from orchestrator job names to avoid triple-nesting (`cd / lint / lint`) in GitHub Actions UI. Mapping: `_tag.yml` → `bump`, `_docker.yml` → `push`, `_deploy.yml` → `run`.
- Pin all `uses:` references to a version tag. Verify the latest with:

  ```bash
  gh release view --repo owner/action --json tagName -q .tagName
  ```

- `homelab-repo` defaults to `Skesov/homelab`. `manifest-path` is always required — it differs per project.

## Validation

Run both before pushing any workflow change:

```bash
# YAML syntax and style
docker run --rm -v $(pwd):/repo cytopia/yamllint:latest -c /repo/.yamllint.yml /repo/.github/workflows/

# GitHub Actions semantics
docker run --rm -v $(pwd):/repo rhysd/actionlint:latest -color /repo/.github/workflows/
```

CI runs both automatically on push and pull request via `validate.yml`.
