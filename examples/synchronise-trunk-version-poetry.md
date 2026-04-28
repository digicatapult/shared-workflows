# Synchronising all PR versions (Poetry)

## Using [synchronise-trunk-version-poetry.yml](../.github/workflows/synchronise-trunk-version-poetry.yml) in callers

This workflow finds all open PRs with one of the labels `v:major`, `v:minor`, or `v:patch` and calls the single-PR synchronisation workflow for each.

### Explicit permissions

```yaml
jobs:
  synchronise-trunk-version-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-trunk-version-poetry.yml@main
    permissions:
      contents: write
      pull-requests: write
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```
