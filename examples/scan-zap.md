# ZAP Scan

## Using [scan-zap.yml](../.github/workflows/scan-zap.yml) in callers

The `matrix_scans` input accepts a JSON array of ZAP scan types to run. Valid values are `baseline`, `full`, `api`, and `automation-framework`. Each scan type runs as a separate parallel job.

### Minimal baseline scan against a docker-compose stack

The most common case: spin up the application stack via a `docker_compose_file`, then run a passive baseline scan against it. Scan results are uploaded as a workflow artifact (`zap_scan-baseline`).

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
    with:
      docker_compose_file: docker-compose.yml
      target: "http://localhost:3000"
```

### Multiple scan types in parallel

Pass multiple scan types as a JSON array. Each runs in its own job, with artifacts named `zap_scan-baseline` and `zap_scan-api` respectively.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
    with:
      matrix_scans: '["baseline", "api"]'
      target: "http://localhost:3000/openapi.json"
      api_scan_format: "openapi"
      docker_compose_file: docker-compose.yml
```

### Pre-scan command to start the application

If the application is started directly on the runner (rather than via docker-compose), use `pre_scan_command` to launch it in the background. The workflow will wait up to `target_wait_timeout` seconds for `target` to respond before starting the scan.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
    with:
      pre_scan_command: "npm ci && npm run build && npm start &"
      target: "http://localhost:3000"
      target_wait_timeout: 120
```

### Full scan with issue writing enabled

The `full` scan type performs active scanning (attacks). Enable `allow_issue_writing` to have ZAP create or update a GitHub issue in the repository with its findings. This requires `issues: write`.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
      issues: write
    with:
      matrix_scans: '["full"]'
      target: "http://localhost:3000"
      docker_compose_file: docker-compose.yml
      allow_issue_writing: true
```

### Automation Framework scan with a custom plan

The `automation-framework` type does not use `target`; instead it requires an `automation_plan` file path checked in to the repository. The wait-for-URL step is skipped for this scan type.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
    with:
      matrix_scans: '["automation-framework"]'
      automation_plan: ".zap/plan.yml"
      docker_compose_file: docker-compose.yml
```

### Pulling images from GHCR

If the application stack pulls images from GitHub Container Registry, set `pull_ghcr: true` to authenticate before bringing up the stack. Add `packages: read` to the permissions block.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
      packages: read
    with:
      pull_ghcr: true
      docker_compose_file: docker-compose.yml
      target: "http://localhost:3000"
```

### Suppressing known alerts with a rules file

Check a ZAP rules TSV file into the repository and pass its path via `rules_file_name` to suppress known false positives.

```yaml
jobs:
  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    permissions:
      contents: read
    with:
      docker_compose_file: docker-compose.yml
      target: "http://localhost:3000"
      rules_file_name: ".zap/rules.tsv"
```
