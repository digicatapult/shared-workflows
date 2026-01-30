# Secret scanning

## Using [scan-secrets.yml](../.github/workflows/scan-secrets.yml) in callers

> [!TIP]
> For public repos, no extra job/workflow permissions are required by any of the options.

### Explicit permissions with defaults

This call invokes the TruffleHog action, scanning for exposed secrets. TruffleHog merely scans the working branch by default. Its results are printed, rather than uploaded.

```yaml
jobs:
  scan-secrets:
    uses: digicatapult/shared-workflows/.github/workflows/scan-secrets.yml@main
    permissions:
      contents: read
```

### Implicit permissions

```yaml
jobs:
  scan-secrets:
    uses: digicatapult/shared-workflows/.github/workflows/scan-secrets.yml@main
```
