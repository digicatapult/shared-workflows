# Vulnerability scanning

## Using [scan-vulns.yml](../.github/workflows/scan-vulns.yml) in callers

`security-events: write`, `contents: read`, and `actions: read` permissions are used. They are invoked at the workflow level for the `semgrep` job.

### Explicit permissions with defaults

As part of this workflow, Semgrep creates a SARIF report that captures any vulnerabilities found. If the repository doesn't have GitHub Code Scanning enabled beforehand, then the required endpoints to POST the report to can't be reached and the workflow itself will likely fail at that point.

```yaml
jobs:
  scan-vulns:
    uses: digicatapult/shared-workflows/.github/workflows/scan-vulns.yml@main
    permissions:
      security-events: write
      contents: read
      actions: read
```
