# Tests

## Using [tests-npm.yml](../.github/workflows/tests-npm.yml) in callers

The permission `pull-requests: write` is assumed at the workflow level the `coverage` job, specifically by the step using `davelosert/vitest-coverage-report-action`. No permissions are required for the `setup` and `tests` jobs.

### Explicit permissions with defaults

This caller workflow invokes the default test scripts, as defined in the `package.json` file for the NPM project.

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
    permissions:
      pull-requests: write
      contents: read
```

### Implicit permissions

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
```
