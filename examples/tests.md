# Tests

## Using [tests-npm.yml](../.github/workflows/tests-npm.yml) in callers

<!-- Verify that pull-requests: write is needed anywhere -->

The permission `pull-requests: write` is assumed at the workflow level for the `setup`, `tests`, and `coverage` jobs.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

This caller workflow invokes the default test scripts, as defined in the `package.json` file for the NPM project.

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
```


### Explicit permissions

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-npm.yml@main
    permissions:
      pull-requests: write
```
