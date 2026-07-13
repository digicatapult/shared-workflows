# Security Scorecard analysis

## Using [generate-security-scorecard.yml](../.github/workflows/generate-security-scorecard.yml) in callers

`security-events: write` and `id-token: write` permissions are used. They are invoked at the workflow level for the `analysis` job.

### Default scan (full check suite, results published)

Runs the complete `ossf/scorecard-action` check suite and uploads results to GitHub Code Scanning. `publish_results: true` also submits results to the public [Scorecard REST API](https://api.scorecard.dev) and enables the Scorecard badge. These benefits are only available in this default mod, using `ossf/scorecard-action`, and cannot be achieved with the CLI.

```yaml
jobs:
  generate-security-scorecard:
    uses: digicatapult/shared-workflows/.github/workflows/generate-security-scorecard.yml@main
    permissions:
      security-events: write
      id-token: write
    with:
      publish_results: true
      upload_type: dashboard
    secrets: inherit
```

### Custom scan (restricted check set)

Runs only the listed checks via the Scorecard CLI directly, instead of the full suite. Useful when only a subset of checks (e.g. branch protection) is relevant to a repository, without the noise of unrelated findings.

```yaml
jobs:
  generate-security-scorecard:
    uses: digicatapult/shared-workflows/.github/workflows/generate-security-scorecard.yml@main
    permissions:
      security-events: write
      id-token: write
    with:
      custom_scan: true
      custom_checks: "Branch-Protection,Code-Review"
      upload_type: dashboard
    secrets: inherit
```

> [!IMPORTANT]
> `publish_results` has no effect when `custom_scan` is `true`. Public transparency (submission to the Scorecard REST API) and the Scorecard README badge are both tied to the full `ossf/scorecard-action` check suite and can't be reimplemented for a partial, custom-checks run. If you need either of those, use the default (`custom_scan: false`) path instead.
