# Synchronising a single PR (Poetry)

## Using [synchronise-pr-version-poetry.yml](../.github/workflows/synchronise-pr-version-poetry.yml) in callers

Several permissions are assumed by this workflow:

- contents: write
- pull-requests: write

### Explicit permissions

This caller synchronises `pyproject.toml` if a version label is applied to the pull request. It requires the secrets `BOT_ID` and `BOT_KEY` as variables at the repository or organisation level.

```yaml
jobs:
  synchronise-pr-version-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/synchronise-pr-version-poetry.yml@main
    permissions:
      contents: write
      pull-requests: write
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      bot-id: ${{ secrets.BOT_ID }}
      bot-key: ${{ secrets.BOT_KEY }}
```
