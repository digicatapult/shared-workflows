# Secret scanning

## Using [scan-secrets.yml](../.github/workflows/scan-secrets.yml) in callers

> [!TIP]
> No extra job/workflow permissions are required by any of the options.


### Implicit permissions with defaults

This call invokes the TruffleHog action, scanning for exposed secrets. TruffleHog merely scans the working branch by default. Its results are printed, rather than uploaded.

```yaml
jobs:
  scan-secrets:
    uses: digicatapult/shared-workflows/.github/workflows/scan-secrets.yml@main
```


### Explicit permissions

```yaml
jobs:
  scan-secrets:
    uses: digicatapult/shared-workflows/.github/workflows/scan-secrets.yml@main
    permissions: {}
```
