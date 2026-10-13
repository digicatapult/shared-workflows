# Post-quantum cryptography scanning

## Using [scan-post-quantum-cryptography.yml](../.github/workflows/scan-post-quantum-cryptography.yml) in callers

Each scan job only needs `contents: read`. The `pqc-report` job also needs `security-events: write` to upload the correlated SARIF report, so GitHub code scanning must be enabled on the repository when `upload_type` is `sarif`.

### Minimal

Runs cryptoscan and Semgrep (pqc-rules), correlates them, and uploads corroborated findings to code scanning.

```yaml
jobs:
  scan-pqc:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      contents: read
      security-events: write
```

### With CodeQL inventory and TLS analysis

CodeQL adds a third source for Python and Java. TLS analysis probes a service started inside the job; targets are `host:port` and must be `localhost` or `127.x.x.x`.

```yaml
jobs:
  scan-pqc:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      contents: read
      packages: read
      security-events: write
    with:
      enable_codeql_inventory: true
      codeql_languages: '["python", "java"]'
      enable_tls_analysis: true
      docker_compose_file: docker-compose.yml
      pull_ghcr: true
      pre_scan_command: npm ci && npm run start:tls &
      tls_targets: '["localhost:3443"]'
      tls_policy: cnsa-2.0-2027
```

### Artefacts only

Keep the reports out of code scanning while evaluating the results.

```yaml
jobs:
  scan-pqc:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      contents: read
    with:
      upload_type: artefact
```

## Creating a baseline

cryptoscan can suppress findings that already exist so that only new ones are reported. Generate the baseline once on `main` and commit it:

```sh
cryptoscan scan . --format json --output baseline.json
git add baseline.json
```

The workflow uses `baseline.json` automatically when it exists; set `scan_baseline_file` to use another path. The baseline only filters cryptoscan, so Semgrep and CodeQL findings in baselined code can still appear, but only as single-source findings in the artefacts.

## How findings reach code scanning

Findings from different tools are merged when they are in the same file, within `correlation_line_window` lines (default 3), and in the same algorithm family. A merged finding is uploaded when:

- two or more sources agree on it (`corroborated`, reported as a warning), or
- it matches the allowlist (reported as a note).

The default allowlist always reports committed private keys (`cryptoscan:KEY-00*`) and localhost TLS results (`tls-analyzer:tls.certificate-key`, `tls-analyzer:tls.negotiated-key-exchange`, `tls-analyzer:tls.legacy-protocol`). Add more with `allowlist_extra`; use `allowlist_disable: true` to report corroborated findings only.

```yaml
with:
  allowlist_extra: "semgrep:pqc.javascript.jwt-*,semgrep:pqc.python.jwt-*"
```

Every finding, reported or not, is in `pqc-all.json` and `pqc-summary.md` in the `pqc-report` artefact.
