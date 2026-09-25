# Applying the version after merge

## Using [apply-version-npm.yml](../.github/workflows/apply-version-npm.yml) in callers

This is the other half of the merge queue versioning model. [require-version-label.yml](require-version-label.md) checks that a pull request declares *what kind* of bump it wants; this workflow computes and applies the actual number on trunk once the pull request has merged.

It reads the version labels of every pull request contained in the push, takes the strongest one (major, then minor, then patch), increments trunk's `package.json` by that amount and pushes a version commit back to trunk. A merge queue can land several pull requests in a single push, which is why the whole push is inspected rather than just the head commit. A push with no labelled pull request produces no bump and therefore no release.

The arithmetic is the same as [synchronise-pr-version-npm.yml](../.github/workflows/synchronise-pr-version-npm.yml). The difference is *where* it runs: on trunk after merge rather than on the pull request branch before it. That is what removes the collision, because no absolute version is ever written to a pull request branch.

### Why this needs the App token, and why the release runs twice

`build-docker`, `generate-sbom` and `release-github` all read the version out of the checked out tree. So the commit that gets built and released has to be the commit that carries the new version.

Pushes made with `GITHUB_TOKEN` deliberately do not trigger workflows. If this workflow used it, the version commit would land but nothing would ever run against it, the release would still be running on the pre-bump commit, `check-version` would report `is_new_version=false`, and no image tag or release would be published. Using the App token means the version commit triggers a second run, and that second run is the one that releases.

So a merge produces two runs on trunk:

1. The merge commit. `apply-version` runs and pushes the version commit. The release jobs are skipped.
2. The version commit. `apply-version` is skipped by the loop guard. The release jobs run, and now see the new version.

The guard is the commit message prefix, and it is what stops the two runs triggering each other forever. Both workflows must agree on it; the default is `chore(release):`.

This is the same GitHub App the `synchronise-*-version` workflows already use, so a repository adopting this needs no new app and no new secrets. It does need the App added to the bypass list of the ruleset protecting trunk, because it now pushes there rather than to a feature branch.

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
    uses: digicatapult/shared-workflows/.github/workflows/apply-version-npm.yml@main
    permissions: {}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```

### Releasing only the version commit

The repository's existing `release.yml` keeps all of its jobs. It gains a single condition so that it releases the version commit rather than the merge commit.

```yaml
name: Release

on:
  push:
    branches: ['main']

jobs:
  static-checks-npm:
    if: ${{ startsWith(github.event.head_commit.message, 'chore(release):') }}
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-npm.yml@main
    permissions:
      security-events: write

  # ...every other release job carries the same condition
```

### Custom trunk branch and commit prefix

```yaml
jobs:
  apply-version:
    if: ${{ !startsWith(github.event.head_commit.message, 'release:') }}
    uses: digicatapult/shared-workflows/.github/workflows/apply-version-npm.yml@main
    permissions: {}
    with:
      trunk-branch: trunk
      release-commit-prefix: 'release:'
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```

## Inputs

| Name                    | Required | Type   | Default             | Description                                                          |
| ----------------------- | -------- | ------ | ------------------- | -------------------------------------------------------------------- |
| `trunk-branch`          | No       | string | `main`              | Branch the version commit is pushed to.                              |
| `release-commit-prefix` | No       | string | `chore(release):`   | Prefix of the version commit. Must match the caller's loop guard.    |

## Outputs

| Name      | Description                                                       |
| --------- | ----------------------------------------------------------------- |
| `bumped`  | `true` when a version commit was pushed.                          |
| `version` | Version written to `package.json`, empty when nothing was bumped. |

## Permissions

None. The workflow authenticates entirely with the App token, so the caller can pass `permissions: {}`.

## Secrets

| Name      | Required | Description                     |
| --------- | -------- | ------------------------------- |
| `bot-id`  | Yes      | GitHub App ID.                  |
| `bot-key` | Yes      | GitHub App private key.         |
