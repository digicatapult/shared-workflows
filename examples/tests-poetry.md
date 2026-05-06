# Tests (Poetry)

## Using [tests-poetry.yml](../.github/workflows/tests-poetry.yml) in callers

`contents: read` is required by the `tests` job to checkout the repository. `packages: read` is required by the `tests` job when `inputs.pull_ghcr` is true.

Note: `docker_compose_file` defaults to `docker-compose.yml`. If your repository does not have a compose file (or you don't want compose to run), set `docker_compose_file: ''`.

### Explicit permissions

```yaml
jobs:
  tests-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/tests-poetry.yml@main
    permissions:
      contents: read
    with:
      docker_compose_file: ''
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
