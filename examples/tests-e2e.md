# End-to-end tests

## Using [tests-e2e-npm.yml](../.github/workflows/tests-e2e-npm.yml) in callers

### Explicit permissions with defaults

This caller invokes the default end-to-end test scripts, as defined in the `package.json` file for the NPM project.

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-npm.yml@main
    permissions:
      contents: read
```

### Minimal with secret inheritance

If the end-to-end tests need to handle tokens or credentials from an earlier workflow, then they can inherit them with the following example:

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-npm.yml@main
    permissions:
      contents: read
    secrets: inherit
```
