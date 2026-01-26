# Version checking

## Using [check-version.yml](../.github/workflows/check-version.yml) in callers

> [!TIP]
> No extra job/workflow permissions are required by any of the options.


### Explicit permissions with defaults

A minimal workflow will check the versions for the working branch in `package*.json`, as the default option is to assume NPM as the package manager. Cargo and Poetry are also supported.

```yaml
jobs:
  check-version:
    uses: digicatapult/shared-workflows/.github/workflows/check-version.yml@main
    permissions: {}
```


### Implicit permissions

```yaml
jobs:
  check-version:
    uses: digicatapult/shared-workflows/.github/workflows/check-version.yml@main
```
