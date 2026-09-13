# AGENTS.md

Shared reusable GitHub Actions workflows for projects under `Skesov/`. Workflows are called from other repositories via `uses: Skesov/github-workflows/.github/workflows/<file>.yml@master`.

## Architecture

Two layers:

**Orchestrators** (`ci.yml`, `cd.yml`) — compose primitives into full pipelines. External repos call these first.

**Primitives** (`_*.yml`) — single-purpose reusable workflows: `_lint-python`, `_test-python`, `_lint-node`, `_test-node`, `_lint-go`, `_test-go`, `_docker`, `_release`, `_deploy`. The underscore prefix signals internal use, but callers do use `_docker.yml` and `_deploy.yml` directly for staging builds and manual redeploys.

`cd.yml` job graph:

```text
changes → lint-<stack> / test-<stack> (matrix per stack) → release → transform → build (matrix) → deploy (matrix)
```

`changes` converts the caller's `components` list into a dorny/paths-filter config, then emits one lint matrix and one test matrix per stack, holding only the components whose `paths` changed. Unknown `stack` values fail the job with an explicit error.

`release` runs `googleapis/release-please-action@v5` after lint and test pass.
On a typical push it only opens or updates the release PR — `transform`/`build`/`deploy`
are skipped. When a release PR is merged, the action creates per-component tags and
GitHub Releases, and `transform` joins those outputs with the caller's `components`
spec on `package-path` to build the matrix that drives `build` and `deploy`.

Gating `release` on lint and test prevents tag creation against a broken master.

The `deploy` matrix uses `max-parallel: 1` to serialize commits against `Skesov/homelab`.

## Conventions

- Primitive job names differ from orchestrator job names to avoid triple-nesting (`cd / lint / lint`) in GitHub Actions UI. Mapping: `_release.yml` → `run`, `_docker.yml` → `push`, `_deploy.yml` → `run`.
- Pin all `uses:` references to a version tag. Verify the latest with:

  ```bash
  gh release view --repo owner/action --json tagName -q .tagName
  ```

- Component YAML is transformed with jq, not yq expressions: the runner ships mikefarah yq, which has no `add` function. `yq -o=json` converts, `jq` transforms. JSON is valid YAML, so dorny/paths-filter accepts the jq output directly.
- Matrix and transform jq programs live in step-level `env:` blocks (`MATRIX_JQ`, `VALIDATE_JQ`, `BUILD_MATRIX_JQ`) so shell quoting stays out of the way. Keep them there when editing.
- `homelab-repo` defaults to `Skesov/homelab`. `deploy.manifest-path` is required per component.
- Every component links to a release-please package via `package-path`. `transform` fails loudly when a release matches no component — silence there once meant a tag with no image.
- `_docker.yml` runs on `ubuntu-24.04-arm` and builds `linux/arm64` by default — matches the arm-only target cluster. Override `runner`/`platforms` per call to build for amd64 (cross-compiled via QEMU).

- **Every job downstream of a matrix-driven lint/test job needs a status function in `if`.** A repo that uses only one stack leaves the other stacks' matrices empty, those jobs are skipped, and GitHub skips every descendant whose `if` lacks `!cancelled()` / `always()` / `success()` — no matter what the condition itself evaluates to. This is why `transform`, `build` and `deploy` all carry `!cancelled() && needs.<prev>.result == 'success'`. Dropping it makes the release tail disappear with no error anywhere.

## Caller requirements

Every consumer of `cd.yml` must:

- Pass `components:` — a YAML list with `name`, `stack`, `paths`, `package-path`, `image` and `deploy` per releasable unit (schema in README)
- Commit `release-please-config.json` and `.release-please-manifest.json` at the repo root, with keys matching every `package-path`
- Grant `permissions: pull-requests: write` (in addition to the existing `contents: write` and `packages: write`) on the calling job
- Pass `secrets:` explicitly — `secrets: inherit` does not match the hyphenated `registry-token`/`gitops-token` parameter names
- Enable "Allow GitHub Actions to create and approve pull requests" in repo settings — release-please cannot open its release PR otherwise

Changing the `components` schema is a breaking change for every caller. Current callers: `Skesov/skesov-bot`, `Skesov/sub-manager-bot`.

## Validation

Run both before pushing any workflow change:

```bash
# YAML syntax and style
docker run --rm -v "$(pwd):/repo" cytopia/yamllint:latest -c /repo/.yamllint.yml /repo/.github/workflows/

# GitHub Actions semantics
docker run --rm -v "$(pwd):/repo" --workdir /repo rhysd/actionlint:latest -color
```

CI runs both automatically on push and pull request via `validate.yml`.

Component transforms can be checked without a runner: extract `MATRIX_JQ` /
`BUILD_MATRIX_JQ` from `cd.yml` with yq and run them against a caller's
`components` block with jq.
