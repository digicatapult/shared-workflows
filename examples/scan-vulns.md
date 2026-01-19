# Vulnerability scanning

## Using [scan-vulns.yml](../.github/workflows/scan-vulns.yml) in callers

Only the `security-events:write` permission is used and it's invoked at the workflow level for the `semgrep` job.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

As part of this workflow, Semgrep creates a SARIF report that captures any vulnerabilities found. If the repository doesn't have GitHub Code Scanning enabled beforehand, then the required endpoints to POST the report to can't be reached and the workflow itself will likely fail at that point.

```yaml
jobs:
  scan-vulns:
    uses: digicatapult/shared-workflows/.github/workflows/scan-vulns.yml@main
```


### Explicit permissions

```yaml
jobs:
  scan-vulns:
    uses: digicatapult/shared-workflows/.github/workflows/scan-vulns.yml@main
    permissions:
      security-events: write
```
