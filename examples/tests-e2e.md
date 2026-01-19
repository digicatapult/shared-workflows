# End-to-end tests

## Using [tests-e2e-npm.yml](../.github/workflows/tests-e2e-npm.yml) in callers

<!-- Verify that contents: write is needed -->

The permission `contents: write` is assumed for the `e2e-tests` job.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

This caller invokes the default end-to-end test scripts, as defined in the `package.json` file for the NPM project.

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-npm.yml@main
```


### Explicit permissions

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-npm.yml@main
    permissions:
      contents: write
```


### Minimal with secret inheritance

If the end-to-end tests need to handle tokens or credentials from an earlier workflow, then they can inherit them with the following example:

```yaml
jobs:
  tests-e2e-npm:
    uses: digicatapult/shared-workflows/.github/workflows/tests-e2e-npm.yml@main
    secrets: inherit
```
