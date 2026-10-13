# Post-quantum cryptography scanning

## Using [scan-post-quantum-cryptography.yml](../.github/workflows/scan-post-quantum-cryptography.yml) in callers

Callers must grant `contents: read` and `security-events: write`. Only the `pqc-report` job uses `security-events: write`, to upload the correlated SARIF report; GitHub code scanning must be enabled on the repository when `upload_type` is `sarif`.

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

### Artefacts only

Keep the reports out of code scanning while evaluating the results. `security-events: write` is still required: GitHub checks a called workflow's job permissions before the run starts, whatever `upload_type` is set to.

```yaml
jobs:
  scan-pqc:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      contents: read
      security-events: write
    with:
      upload_type: artefact
```

## Creating a baseline

cryptoscan can suppress findings that already exist so that only new ones are reported. Generate the baseline once on `main` and commit it:

```sh
cryptoscan scan . --format json --output baseline.json
git add baseline.json
```

The workflow uses `baseline.json` automatically when it exists; set `scan_baseline_file` to use another path. The baseline only filters cryptoscan, so Semgrep findings in baselined code become single-source: they stay in the artefacts but no longer reach code scanning.

## How findings reach code scanning

Findings from different tools are merged when they are in the same file, within `correlation_line_window` lines (default 3), and in the same algorithm family. A merged finding is uploaded when:

- both sources agree on it (`corroborated`, reported as a warning), or
- it matches the allowlist (reported as a note).

The default allowlist always reports committed private keys (`cryptoscan:KEY-00*`), which Semgrep cannot detect. Add more with `allowlist_extra`; use `allowlist_disable: true` to report corroborated findings only.

```yaml
with:
  allowlist_extra: "semgrep:pqc.javascript.jwt-*,semgrep:pqc.python.jwt-*"
```

Every finding, reported or not, is in `pqc-all.json` and `pqc-summary.md` in the `pqc-report` artefact.
