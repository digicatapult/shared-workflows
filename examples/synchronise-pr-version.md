# Synchronising a single PR

## Using [synchronise-pr-version-npm.yml](../.github/workflows/synchronise-pr-version-npm.yml) in callers

<!-- Verify that contents: write is needed -->

Several permissions are assumed by this workflow:
- contents: write
- pull-requests: write

The permission `pull-requests:write` is required by the third-party action `planetscale/ghcommit-action` as part of the `synchronise-version` job. It's assumed at the workflow level.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

This caller synchronises `package.json` if a given label is applied to the pull request. It requires the secrets `BOT_ID` and `BOT_KEY` as variables at the repository or organisation level.

```yaml
jobs:
  synchronise-pr-version-npm:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-pr-version-npm.yml@main
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```


### Explicit permissions

```yaml
jobs:
  synchronise-pr-version-npm:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-pr-version-npm.yml@main
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
    permissions:
      contents: write
      pull-requests: write
```
