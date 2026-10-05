# Setup the Poetry environment

Composite action that installs Python and Poetry, then restores the Poetry virtualenv from the Actions cache or
creates it with `poetry install`. Imported from
[`networktocode/gh-action-setup-poetry-environment`](https://github.com/networktocode/gh-action-setup-poetry-environment)
at `v7.2`; that repository is unchanged and keeps serving its existing tags.

```yaml
- name: "Setup environment"
  uses: "networktocode/github-reusable-workflows/.github/actions/setup-poetry-environment@v1"
  with:
    poetry-version: "2.1.3"
    python-version: "3.12"
    poetry-install-options: "--only dev"
```

The cache key is:

```text
${{ runner.os }}-poetry-${{ steps.python-setup.outputs.python-version }}-${{ hashFiles('./poetry.lock') }}${{ inputs.cache-version }}
```

A changed `poetry.lock`, a different exact Python version or a new `cache-version` invalidates the cache and a fresh
environment is installed and saved.

## Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `python-version` | `"3.11"` | Python version passed to `actions/setup-python`. |
| `poetry-version` | `""` (latest) | Poetry version installed with `pipx`, e.g. `2.1.3`. Versions `1.x` also get a compatible `virtualenv` plugin. |
| `poetry-install-options` | `"--only dev"` | Options appended to `poetry install`. |
| `cache-version` | `""` | Extra string appended to the cache key to force a fresh environment. |

## Outputs

| Output | Purpose |
| --- | --- |
| `cache-used` | `true` when the virtualenv was restored from cache, empty otherwise. |
| `python-version` | Exact Python version installed, e.g. `3.12.4`. |
