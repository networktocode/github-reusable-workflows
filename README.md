# github-reusable-workflows

Centralized, versioned [GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
for Network to Code projects across the `nautobot`, `networktocode` and `networktocode-llc` organizations:
Nautobot Apps (`cookiecutter-nautobot-app`) and Python libraries such as `pyntc` and `netutils` (`cookiecutter-ntc`).

Every repository keeps a handful of tiny *caller* workflows in its own `.github/workflows/` directory that delegate
to the reusable workflows here. The composite action they all depend on lives here too. Fixing or improving CI then happens once, in this repository, and every project picks
it up on its next run instead of needing the same change copied into dozens of repositories.

The workflows expect the standard NTC project layout: `tasks.py` invoke targets (`ruff`, `yamllint`, `pylint`,
`lock`, `build-and-check-docs`, `generate-release-notes`, plus `unittest`/`validate-app-config`/`check-migrations`
for apps or `pytest` for libraries), towncrier fragments in `changes/`, `docs/admin/release_notes/`, and a
`Dockerfile` (`development/Dockerfile` for apps, `./Dockerfile` for libraries).

## Layout

```text
.github/actions/
  setup-poetry-environment/           composite action: Python + Poetry + cached virtualenv (used by every workflow)
.github/workflows/
  ntc-ci.yml                          reusable: lint, docs, lock check, docker checks, tests, coverage, changelog
  ntc-coverage.yml                    reusable: post the coverage comment to the PR (workflow_run)
  ntc-prepare-release.yml             reusable: bump version, release notes, release PR, draft GitHub release
  ntc-release.yml                     reusable: build, publish to PyPI/Artifactory, Slack, post-release PRs
  ntc-nautobot-upstream-testing.yml   reusable (apps only): test against in-development Nautobot branches
  _ci.yml                             this repository's own CI (yamllint + actionlint)
  _tag-major.yml                      moves the floating vMAJOR tag on each release of this repository
examples/
  nautobot-app/                       drop-in callers for open source Nautobot apps
  nautobot-app-commercial/            drop-in callers for commercial Nautobot apps (Artifactory)
  python-library/                     drop-in callers for Python libraries (pyntc, netutils, ...)
```

## Adopting in a repository

1. Delete the existing `ci.yml`, `coverage.yml`, `prepare_release.yml`, `release.yml` and `upstream_testing.yml`
   from the repository's `.github/workflows/`.
2. Copy the files from the matching `examples/` directory into `.github/workflows/`, **keeping the same file
   names**. PyPI Trusted Publishing is bound to the caller workflow's file name (`release.yml`) and the `pypi`
   environment, so nothing changes on the PyPI side.
3. Edit the caller inputs: `project_type`, `project_name`, `min_nautobot_version` (apps), the `push.branches`
   list, `changelog_base_branches` (add `ltm-*` branches that accept feature PRs), `python_versions` /
   `lint_checks` (libraries) and the upstream testing matrix (apps).
4. Update branch protection / rulesets: the required status check names change (see
   [Status check names](#status-check-names)).
5. Keep `CODEOWNERS`, `dependabot.yml`, `ISSUE_TEMPLATE/` and `pull_request_template.md` in the repository.
   These are not workflows and cannot be centralized here; see [What stays per-repo](#what-stays-per-repo).

Minimal callers (`.github/workflows/ci.yml`):

```yaml
# Nautobot app
jobs:
  ci:
    uses: "networktocode/github-reusable-workflows/.github/workflows/ntc-ci.yml@v1"
    permissions:
      contents: "write"
      pull-requests: "write"
    with:
      project_type: "nautobot-app"
      project_name: "nautobot-golden-config"
      min_nautobot_version: "3.1.0"
```

```yaml
# Python library
jobs:
  ci:
    uses: "networktocode/github-reusable-workflows/.github/workflows/ntc-ci.yml@v1"
    with:
      project_type: "python-library"
      project_name: "pyntc"
      python_versions: '["3.10", "3.11"]'
```

## Reusable workflows

### `ntc-ci.yml`

Runs on `push` / `pull_request` (triggers live in the caller). Jobs: `config`, `prepare-lockfile`, `lint`
(matrix), `check-docs-build`, `poetry`, `check-in-docker` (pylint, plus app config validation and migrations
for apps), `unittest` (matrix), `unittest_report` (apps: coverage on the newest Python), `changelog` (towncrier
fragment check on feature PRs).

What `project_type` changes:

| | `nautobot-app` | `python-library` |
| --- | --- | --- |
| Lockfile for lint jobs | regenerated (`poetry lock --regenerate`) | committed `poetry.lock` |
| Default lint checks | ruff-format, ruff-lint, djlint, yamllint, markdownlint | ruff-format, ruff-lint, yamllint |
| Docker checks | `development/Dockerfile` on min Python / min Nautobot | `./Dockerfile` on each of `python_versions` |
| Tests | `invoke unittest` inside the image, postgres and mysql matrix | `invoke pytest` on the host, one job per Python version |
| Coverage comment | yes | no |

| Input | Default | Purpose |
| --- | --- | --- |
| `project_type` | required | `nautobot-app` or `python-library`. |
| `project_name` | required | Distribution / image name, e.g. `nautobot-golden-config`, `pyntc`. |
| `invoke_context_name` | derived | `INVOKE_<NAME>_*` env var prefix. Defaults to `project_name` upper-cased with `-` replaced by `_`. |
| `min_nautobot_version` | `""` | Apps (required): lowest supported Nautobot version. |
| `min_python_version` | `"3.10"` | Lowest supported Python. |
| `max_python_version` | `"3.14"` | Highest supported Python. |
| `python_versions` | `[min, max]` | Libraries: JSON list of Python versions for the docker checks and pytest matrices. |
| `poetry_version` | `"2.1.3"` | Poetry version on the runners. |
| `regenerate_lockfile` | by type | `"true"` / `"false"`: lint against a regenerated lock or the committed one. |
| `lint_checks` | by type | JSON list of invoke tasks to run as lint jobs (`ruff-format` / `ruff-lint` map to `ruff --action ...`). Add `mypy` for typed libraries. |
| `check_docs` | `true` | Run `invoke build-and-check-docs`. |
| `unittest_matrix` | by type | JSON `strategy.matrix` override for the test job. |
| `unittest_runner` | `"ubuntu-latest"` | Runner label for the test job. |
| `changelog_base_branches` | `["develop"]` | JSON list of PR base branches that require a changelog fragment and get a coverage comment. |
| `coverage_minimum_green` | `90` | Apps: coverage comment green threshold. |
| `coverage_minimum_orange` | `80` | Apps: coverage comment orange threshold. |

| Secret | Required | Purpose |
| --- | --- | --- |
| `ARTIFACTORY_USERNAME` / `ARTIFACTORY_PASSWORD` | no | Commercial projects: credentials for the private `artifactory-pypi` Poetry source, also passed to the docker build as BuildKit secrets. |

Caller permissions: apps need `contents: write` and `pull-requests: write` (the coverage action pushes its data
branch and prepares the PR comment). Libraries need only the defaults.

### `ntc-coverage.yml`

Triggered by the caller on `workflow_run` of the `CI` workflow. Downloads the comment artifact produced by the CI
run and posts or updates the coverage comment on the PR. No checkout is performed (untrusted PR code never runs
with write permissions). Caller permissions: `pull-requests: write`, `contents: write`, `actions: read`. Only
apps produce the artifact today, so libraries do not need this caller.

### `ntc-prepare-release.yml`

Triggered by the caller on `workflow_dispatch`; the caller forwards the dispatch inputs.

| Input | Default | Purpose |
| --- | --- | --- |
| `bump_rule` | required | `prerelease`, `patch`, `minor` or `major`. |
| `target_branch` | `"main"` | Branch the release PR targets. `main` releases branch from `develop`. |
| `date` | today (US Eastern) | Release date, `YYYY-MM-DD`. |
| `previous_version` | latest tag | Previous tag for the generated GitHub release notes. |
| `poetry_version` | `"2.1.3"` | Poetry version on the runner. |

| Secret | Required | Purpose |
| --- | --- | --- |
| `GH_BOT_TOKEN` | no | Bot PAT used to open the release PR (`GH_NAUTOBOT_BOT_TOKEN` for OSS apps, `NTC_LLC_NAUTOBOT_APPS_REPO_WORKFLOW_GH_TOKEN` for commercial). Without it `GITHUB_TOKEN` is used and the PR does **not** trigger CI. |
| `ARTIFACTORY_USERNAME` / `ARTIFACTORY_PASSWORD` | no | Commercial projects only. |

### `ntc-release.yml`

Triggered by the caller on `release: published`. Builds the package (and docs unless `build_docs: false`), checks
the tag matches `pyproject.toml`, uploads the dist files to the GitHub release, publishes to PyPI (Trusted
Publishing) or Artifactory, optionally notifies Slack, then opens the post-release PRs (`main -> develop`,
`main -> next` when a `next` branch exists, `ltm-*` version bump, and `ltm-*` release notes sync into `develop`).

| Input | Default | Purpose |
| --- | --- | --- |
| `poetry_version` | `"2.1.3"` | Poetry version on the runners. |
| `python_version` | `"3.12"` | Python used to build. |
| `publish_target` | `"pypi"` | `pypi` or `artifactory`. |
| `publish_environment` | `"pypi"` | GitHub environment for the publish job; must match the PyPI Trusted Publisher configuration. |
| `artifactory_url` | `""` | Required when `publish_target` is `artifactory` (pass `${{ vars.ARTIFACTORY_URL }}`). |
| `build_docs` | `true` | Run `invoke build-and-check-docs` before building. Libraries built on Read the Docs set `false`. |
| `release_notes_path` | `"docs/admin/release_notes/"` | Directory synced from `ltm-*` releases into `develop`. |

| Secret | Required | Purpose |
| --- | --- | --- |
| `GH_BOT_TOKEN` | no | Bot PAT used to open the post-release PRs. Without it `GITHUB_TOKEN` is used and those PRs do **not** trigger CI. |
| `SLACK_WEBHOOK_URL` | no | Incoming webhook for the release notification (`OSS_PYPI_SLACK_WEBHOOK_URL`). Skipped when unset. |
| `ARTIFACTORY_USERNAME` / `ARTIFACTORY_PASSWORD` | no | Commercial projects: private Poetry source and `poetry publish` credentials. |

Caller permissions: `contents: write`, `pull-requests: write`, `id-token: write`.

### `ntc-nautobot-upstream-testing.yml`

Apps only. Thin wrapper around Nautobot's own reusable
[`plugin_upstream_testing_base.yml`](https://github.com/nautobot/nautobot/blob/develop/.github/workflows/plugin_upstream_testing_base.yml)
so the matrix and the pinned Nautobot workflow ref live in one place.

| Input | Default | Purpose |
| --- | --- | --- |
| `project_name` | required | Distribution name of the app. |
| `invoke_context_name` | derived | As in the CI workflow. |
| `matrix` | develop/main and next/main | JSON list of `{"nautobot_branch": ..., "app_branch": ...}` pairs. |

## Composite actions

### `setup-poetry-environment`

Installs Python and Poetry and restores or creates the cached Poetry virtualenv. Imported from
`networktocode/gh-action-setup-poetry-environment` at `v7.2`; that repository is untouched and its `v7` tags keep
working for existing callers. New callers and every reusable workflow in this repository use:

```yaml
uses: "networktocode/github-reusable-workflows/.github/actions/setup-poetry-environment@v1"
```

Inputs and outputs are documented in [`.github/actions/setup-poetry-environment/README.md`](.github/actions/setup-poetry-environment/README.md).

The reusable workflows reference the action by its full path pinned to the floating major tag (`@v1`), because a
relative `./` path would resolve against the *caller's* checkout. Two consequences:

- The `v1` tag must exist before any caller can run the workflows; publish the first release before migrating repos.
- A change to the action on a branch is not exercised by callers pointed at that branch until it is released, since
  the workflows still fetch the action from `@v1`. Test action changes with the `_ci.yml` smoke job or by temporarily
  pointing the `uses:` lines at the branch.

## Status check names

With reusable workflows GitHub names checks `<caller job> / <reusable job name>`. With the caller job named `ci`
the checks to require are, for example:

- `ci / Linting: ruff-format`, `ci / Linting: ruff-lint`, `ci / Linting: yamllint` (plus `djlint`, `markdownlint` or `mypy` as configured)
- `ci / Check docs build`, `ci / Check poetry.lock`
- `ci / Checks in docker (Python 3.10, Nautobot 3.1.0)` or `ci / Checks in docker (Python 3.10)`
- `ci / Unit tests (Python 3.10, Nautobot stable, postgresql)` or `ci / Unit tests (Python 3.10)` (one per matrix entry)
- `ci / Unit tests with coverage (Python 3.14, Nautobot stable, postgresql)` (apps)
- `ci / Check changelog fragment`

## Behavior differences from the per-repo copies

- The individual lint jobs are one matrix job driven by `lint_checks`; `check-in-docker` waits for all of them.
- The `INVOKE_<PROJECT>_*` variables are written to `GITHUB_ENV` at runtime because YAML keys cannot be templated.
- Library pytest jobs install the matrix Python version on the host. The per-repo copies always ran the host
  tests on the setup action's default interpreter, so the Python matrix was not actually exercised.
- The coverage-comment action pin is the correct 40-character SHA (several repos carry a 41-character typo in
  `coverage.yml`).
- Action pins follow the current cookiecutter templates (`actions/checkout@v7`, buildx v4.3.0, build-push v7.3.0).
- Inputs such as `project_type`, `bump_rule` and `publish_target` are validated up front with clear errors.
- Libraries get the app-style post-release automation for `next` and `ltm-*` branches for free; the jobs simply
  do nothing when those branches do not exist.

## Versioning and testing changes

- Releases are tagged `vMAJOR.MINOR.PATCH`; `_tag-major.yml` moves the floating `vMAJOR` tag. Callers pin `@v1`.
- Breaking changes to inputs, secrets, job names or the composite action bump the major version. When cutting a new
  major, also update the `@v1` action reference inside the reusable workflows.
- To test a change before release, point one repository's caller at a branch: `...ntc-ci.yml@my-branch`.
- Dependabot keeps the pinned third-party actions current in this repository.

## Repository requirements

- Callers in all three organizations can use this repository only if it is **public**, or if it is private and
  *Settings > Actions > General > Access* allows the calling repositories (private cross-organization access
  requires both repositories to be in the same GitHub Enterprise).
- Publishing projects need the `pypi` environment with a PyPI Trusted Publisher for `release.yml`, and optionally
  the `OSS_PYPI_SLACK_WEBHOOK_URL` secret and a bot PAT (organization level is fine).
- Commercial projects need `ARTIFACTORY_USERNAME`, `ARTIFACTORY_PASSWORD`,
  `NTC_LLC_NAUTOBOT_APPS_REPO_WORKFLOW_GH_TOKEN` and the `ARTIFACTORY_URL` variable.

## What stays per-repo

Reusable workflows only cover `jobs`. `CODEOWNERS` and `dependabot.yml` are read only from the repository itself.
Issue and pull request templates can be given organization-wide defaults by adding them to a repository named
`.github` in each organization (`nautobot/.github` already exists); repository-local copies still win when present.
Project-specific automation such as netutils' data file pulls stays in that repository.
