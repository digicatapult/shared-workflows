# Applying the version after merge (Poetry)

## Using [apply-version-poetry.yml](../.github/workflows/apply-version-poetry.yml) in callers

The Poetry counterpart of [apply-version-npm.yml](apply-version.md). It behaves identically, reading and writing the version in `pyproject.toml` with `poetry version` instead of `package.json` with `npm version`. The version is read from `[tool.poetry]` and falls back to `[project]`, matching [synchronise-pr-version-poetry.yml](../.github/workflows/synchronise-pr-version-poetry.yml).

Read [apply-version.md](apply-version.md) for the full explanation of the model: why the App token is required, why a merge produces two runs on trunk, and how the commit prefix guard keeps the two workflows from triggering each other.

### Applying the version

```yaml
name: Apply version

on:
  push:
    branches: ['main']

jobs:
  apply-version:
    # Do not bump on top of a bump. Without this guard the version commit
    # pushed below would trigger this workflow again.
    if: ${{ !startsWith(github.event.head_commit.message, 'chore(release):') }}
    uses: digicatapult/shared-workflows/.github/workflows/apply-version-poetry.yml@main
    permissions: {}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```

### Pinning the Python version

```yaml
jobs:
  apply-version:
    if: ${{ !startsWith(github.event.head_commit.message, 'chore(release):') }}
    uses: digicatapult/shared-workflows/.github/workflows/apply-version-poetry.yml@main
    permissions: {}
    with:
      python-version: "3.12"
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```

## Inputs

| Name                    | Required | Type   | Default           | Description                                                       |
| ----------------------- | -------- | ------ | ----------------- | ----------------------------------------------------------------- |
| `trunk-branch`          | No       | string | `main`            | Branch the version commit is pushed to.                           |
| `python-version`        | No       | string | `3.14`            | Python version used to read `pyproject.toml` and run Poetry.      |
| `release-commit-prefix` | No       | string | `chore(release):` | Prefix of the version commit. Must match the caller's loop guard. |

## Outputs

| Name      | Description                                                        |
| --------- | ------------------------------------------------------------------ |
| `bumped`  | `true` when a version commit was pushed.                           |
| `version` | Version written to `pyproject.toml`, empty when nothing was bumped. |

## Permissions

None. The workflow authenticates entirely with the App token, so the caller can pass `permissions: {}`.

## Secrets

| Name      | Required | Description             |
| --------- | -------- | ----------------------- |
| `bot-id`  | Yes      | GitHub App ID.          |
| `bot-key` | Yes      | GitHub App private key. |
