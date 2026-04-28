# Tests (Poetry)

## Using [tests-poetry.yml](../.github/workflows/tests-poetry.yml) in callers

`contents: read` is required by the `setup` job. `contents: read` is required by the `tests` job, and `packages: read` is required by the `tests` job when `inputs.pull_ghcr` is true.

### Explicit permissions

```yaml
jobs:
  tests-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/tests-poetry.yml@main
    permissions:
      contents: read
      pull-requests: read
```

### With integration dependencies

```yaml
jobs:
  tests-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/tests-poetry.yml@main
    permissions:
      contents: read
      packages: read
    with:
      docker_compose_file: docker-compose.yml
      tests: '["tests/unit", "tests/integration"]'
```
