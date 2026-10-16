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

Every Semgrep finding is kept as its own finding, so distinct calls on neighbouring lines (for example an RSA key and an RS256 JWT) stay separate. A cryptoscan finding corroborates the nearest Semgrep finding in the same file and algorithm family, within `correlation_line_window` lines (default 3). A finding is uploaded when:

- both sources agree on it (`corroborated`, reported as a warning), or
- it matches the allowlist (reported as a note).

The default allowlist reports findings that only one tool can detect, so they could never be corroborated:

- JWT signing with RSA or ECC (`semgrep:pqc.*.jwt-*`)
- legacy TLS versions and classical-only key exchange groups (`semgrep:pqc.*.tls-legacy-version`, `semgrep:pqc.*.tls-classical-groups`)

Add more with `allowlist_extra`; it extends the defaults rather than replacing them.

Committed private keys (`cryptoscan:KEY-00*`) are not reported by default. cryptoscan matches only the PEM header, so it also flags code and tests that handle PEM strings; GitHub secret scanning detects real keys. Add `cryptoscan:KEY-00*` to `allowlist_extra` to report cryptoscan's matches as well.

```yaml
with:
  allowlist_extra: "semgrep:pqc.javascript.webcrypto-*,semgrep:pqc.javascript.forge-*"
```

Every finding, reported or not, is in `pqc-all.json` and `pqc-summary.md` in the `pqc-report` artefact.
