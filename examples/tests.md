# Tests

## Using [tests-npm.yml](../.github/workflows/tests-npm.yml) in callers

`contents: read` is required by the `setup` job. `contents: read` and `packages: read` are required by the `tests` job. Permissions `pull-requests: write` and `contents: read` are required by the `coverage` job (specifically by the step using `davelosert/vitest-coverage-report-action`).

### Explicit permissions with defaults

This caller workflow invokes the default test scripts, as defined in the `package.json` file for the NPM project.

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
    permissions:
      pull-requests: write
      contents: read
      packages: read
```

### Implicit permissions

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
```
