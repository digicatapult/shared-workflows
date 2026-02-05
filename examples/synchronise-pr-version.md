# Synchronising a single PR

## Using [synchronise-pr-version-npm.yml](../.github/workflows/synchronise-pr-version-npm.yml) in callers

Several permissions are assumed by this workflow:

- contents: write
- pull-requests: write

The permission `contents: write` is required by the third-party action `planetscale/ghcommit-action` as part of the `synchronise-version` job, and `pull-requests: write` is then needed to finish processing any labels against the PR. They're both assumed at the workflow level.

### Explicit permissions with defaults

This caller synchronises `package.json` if a given label is applied to the pull request. It requires the secrets `BOT_ID` and `BOT_KEY` as variables at the repository or organisation level.

```yaml
jobs:
  synchronise-pr-version-npm:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-pr-version-npm.yml@main
    permissions:
      contents: write
      pull-requests: write
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```
