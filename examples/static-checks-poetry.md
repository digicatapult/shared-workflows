# Performing static checks (Poetry)

## Using [static-checks-poetry.yml](../.github/workflows/static-checks-poetry.yml) in callers

`static-checks` and `scan-secrets` require `contents: read` permissions, and `scan-vulns` requires `security-events: write`, `contents: read`, and `actions: read`.

### Explicit permissions

```yaml
jobs:
  static-checks-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-poetry.yml@main
    permissions:
      security-events: write
      contents: read
      actions: read
```

### Minimal without Semgrep checks

```yaml
jobs:
  static-checks-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-poetry.yml@main
    permissions:
      contents: read
    with:
      enable_semgrep_action: false
```

### Customising which tools run

By default the workflow runs a matrix of full commands under `poetry run`.

```yaml
jobs:
  static-checks-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-poetry.yml@main
    permissions:
      contents: read
    with:
      matrix_commands: '["pylint app", "black --check .", "mypy app/"]'
```
