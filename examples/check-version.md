# Version checking

## Using [check-version.yml](../.github/workflows/check-version.yml) in callers

<!-- contents:write needs to be verified. -->

The permission `contents:write` is included in this workflow. It's invoked at the workflow level for the `check-version` job.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

A minimal workflow will check the versions for the working branch in `package*.json`, as the default option is to assume NPM as the package manager. Cargo and Poetry are also supported.

```yaml
jobs:
  check-version:
    uses: digicatapult/shared-workflows/.github/workflows/check-version.yml@main
```


### Explicit permissions

```yaml
jobs:
  check-version:
    uses: digicatapult/shared-workflows/.github/workflows/check-version.yml@main
    permissions:
      contents: write
```
