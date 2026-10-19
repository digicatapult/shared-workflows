# Post-quantum cryptography scanning

## Using [scan-post-quantum-cryptography.yml](../.github/workflows/scan-post-quantum-cryptography.yml) in callers

Callers must grant `actions: read`, `contents: read` and `security-events: write`. Only the `pqc-report` job uses `actions: read` and `security-events: write`, to upload the correlated SARIF report; `actions: read` is only needed for that upload in private repositories. GitHub code scanning must be enabled on the repository when `upload_type` is `sarif`.

### Minimal

Runs cryptoscan and Semgrep (pqc-rules), correlates them, and uploads corroborated findings to code scanning.

```yaml
jobs:
  scan-pqc:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      actions: read
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
      actions: read
      contents: read
      security-events: write
    with:
      upload_type: artefact
```

### Calling more than once in a run

Artefact names and the code scanning category must be unique within a workflow run. Give each call its own `artifact_suffix`; it is appended to every artefact name (`pqc-raw-cryptoscan-sarif`, `pqc-report-sarif`, ...) and to the code scanning category (`pqc-sarif`).

```yaml
jobs:
  scan-pqc-sarif:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      actions: read
      contents: read
      security-events: write
    with:
      artifact_suffix: "-sarif"
  scan-pqc-artefact:
    uses: digicatapult/shared-workflows/.github/workflows/scan-post-quantum-cryptography.yml@main
    permissions:
      actions: read
      contents: read
      security-events: write
    with:
      upload_type: artefact
      artifact_suffix: "-artefact"
```

Changing `artifact_suffix` on an existing `sarif` caller moves its results to a new category; alerts under the old category stay open until they are dismissed or the category is deleted.

## Creating a baseline

cryptoscan can suppress findings that already exist so that only new ones are reported. Generate the baseline in CI, not locally: cryptoscan records absolute file paths and matches baseline entries on path, line and pattern, so a baseline generated on a laptop never matches the runner's paths and would commit local paths to the repository.

Run the workflow on `main`, then commit its raw cryptoscan output as the baseline:

```sh
gh run download <run-id> --name pqc-raw-cryptoscan --dir pqc-baseline
mv pqc-baseline/cryptoscan.json baseline.json
git add baseline.json
```

Add your `artifact_suffix` to the artefact name if you set one.

The workflow uses `baseline.json` automatically when it exists; set `scan_baseline_file` to use another path. Because the line number is part of the match, any edit that moves a baselined finding (for example adding lines above it) makes it a new finding again; regenerate the baseline when that becomes noisy. The baseline only filters cryptoscan, so Semgrep findings in baselined code become single-source: they stay in the artefacts but no longer reach code scanning.

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
