# End-to-end tests (Poetry)

## Using [tests-e2e-poetry.yml](../.github/workflows/tests-e2e-poetry.yml) in callers

### Explicit permissions

```yaml
jobs:
  tests-e2e-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-poetry.yml@main
    permissions:
      contents: read
```

### Minimal with secret inheritance

```yaml
jobs:
  tests-e2e-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-poetry.yml@main
    permissions:
      contents: read
    secrets: inherit
```
