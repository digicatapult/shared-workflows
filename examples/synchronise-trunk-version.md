# Synchronising all PRs

## Using [synchronise-trunk-version-npm.yml](../.github/workflows/synchronise-trunk-version-npm.yml) in callers

The following permissions are assumed by the job `find-pull-requests`:

- contents: read
- pull-requests: read

The following permissions are assumed by the job `synchronise-pull-requests`:

- contents: write
- pull-requests: write

Both permissions are required by the upstream [synchronise-pr-version-npm.yml](../.github/workflows/synchronise-pr-version-npm.yml) workflow.

### Explicit permissions with defaults

This caller synchronises the versions of all open PRs. It requires the secrets `BOT_ID` and `BOT_KEY` as variables at the repository or organisation level.

```yaml
jobs:
  synchronise-trunk-version-npm:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-trunk-version-npm.yml@main
    permissions:
      contents: write
      pull-requests: write
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```

### Implicit permissions

```yaml
jobs:
  synchronise-trunk-version-npm:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-trunk-version-npm.yml@main
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```
