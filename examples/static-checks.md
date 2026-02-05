# Performing static checks

## Using [static-checks-npm.yml](../.github/workflows/static-checks-npm.yml) in callers

`static-checks` and `scan-secrets` require `contents: read` permissions, and `scan-vulns` requires `security-events: write`, `contents: read`, and `actions: read`.

### Explicit permissions with defaults

As part of this workflow, Semgrep creates a SARIF report that captures any vulnerabilities found. If the repository doesn't have GitHub Code Scanning enabled beforehand, then the required endpoints to POST the report to can't be reached and the workflow itself will likely fail at that point.

```yaml
jobs:
  static-checks-npm:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-npm.yml@main
    permissions:
      security-events: write
      contents: read
      actions: read
```

### Minimal with PR artefacts

If `inputs.semgrep_upload_type` is left empty (`""`) or set to `artefact`, then the step requiring `security-events: write` is skipped. It's only needed to upload directly to GitHub via the `github/codeql-action/upload-sarif` action.

```yaml
jobs:
  static-checks-npm:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-npm.yml@main
    permissions:
      security-events: write
      contents: read
      actions: read
    with:
      semgrep_upload_type: "artefact"
```

### Minimal without Semgrep checks

If `inputs.enable_semgrep_action` is changed to `false`, then Semgrep isn't executed at all.

```yaml
jobs:
  static-checks-npm:
    uses: digicatapult/shared-workflows/.github/workflows/static-checks-npm.yml@main
    permissions:
      security-events: write
      contents: read
      actions: read
    with:
      enable_semgrep_action: false
```
