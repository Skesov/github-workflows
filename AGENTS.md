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
- `_docker.yml` runs on `ubuntu-24.04-arm` and builds `linux/arm64` by default — matches the arm-only target cluster. Override `runner`/`platforms` per call to build for amd64 (cross-compiled via QEMU).

- **Every job downstream of a matrix-driven lint/test job needs a status function in `if`.** A repo that uses only one stack leaves the other stacks' matrices empty, those jobs are skipped, and GitHub skips every descendant whose `if` lacks `!cancelled()` / `always()` / `success()` — no matter what the condition itself evaluates to. This is why `transform`, `build` and `deploy` all carry `!cancelled() && needs.<prev>.result == 'success'`. Dropping it makes the release tail disappear with no error anywhere.

## Caller requirements

Every consumer of `cd.yml` must:

- Commit `release-please-config.json` and `.release-please-manifest.json` at the repo root
- Grant `permissions: pull-requests: write` (in addition to the existing `contents: write` and `packages: write`) on the calling job
- Pass `secrets:` explicitly — `secrets: inherit` does not match the hyphenated `registry-token`/`gitops-token` parameter names
- Enable "Allow GitHub Actions to create and approve pull requests" in repo settings — release-please cannot open its release PR otherwise

## Validation

Run both before pushing any workflow change:

```bash
# YAML syntax and style
docker run --rm -v "$(pwd):/repo" cytopia/yamllint:latest -c /repo/.yamllint.yml /repo/.github/workflows/

# GitHub Actions semantics
docker run --rm -v "$(pwd):/repo" --workdir /repo rhysd/actionlint:latest -color
```

CI runs both automatically on push and pull request via `validate.yml`.
