# Fallow audit

## Using [fallow-npm.yml](../.github/workflows/fallow-npm.yml) in callers

Runs [Fallow](https://github.com/fallow-rs/fallow) and reports unused code, duplication and complexity as a sticky PR comment, plus inline review comments or annotations. The audit is advisory by default (`fail_on: none`) and does not fail the job. See [Scope and blocking](../README.md#scope-and-blocking) for what each `scope` and `fail_on` value reports and blocks on, and what `scope: changed` cannot see.

Fallow is installed by the workflow at `fallow_version`, so do not add it to the caller's `package.json`. Project-specific settings live in the caller's `.fallowrc.json`, which Fallow discovers automatically.

### Minimal (advisory, changed lines)

```yaml
jobs:
  fallow:
    uses: digicatapult/shared-workflows/.github/workflows/fallow-npm.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
```

### Whole codebase, advisory

```yaml
jobs:
  fallow:
    uses: digicatapult/shared-workflows/.github/workflows/fallow-npm.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
    with:
      scope: all
```

### Block on new error-severity findings

Fails only on findings the pull request introduces that have `error` severity in `.fallowrc.json`; set a rule to `warn` to report it without blocking. Pin `fallow_version` so a new Fallow release cannot start failing pull requests.

```yaml
jobs:
  fallow:
    uses: digicatapult/shared-workflows/.github/workflows/fallow-npm.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
    with:
      fail_on: new
      fallow_version: "3.31.0"
```

### Block on new dead code across the whole codebase

Catches files and exports that a pull request orphans elsewhere in the codebase. Commit a baseline of the existing dead code first, using the same Fallow version as the workflow:

```bash
npx fallow@3.31.0 --only dead-code --save-baseline fallow-baseline.json
```

```yaml
jobs:
  fallow:
    uses: digicatapult/shared-workflows/.github/workflows/fallow-npm.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
    with:
      scope: all
      analyses: dead-code
      baseline: fallow-baseline.json
      fail_on: any
      fallow_version: "3.31.0"
```

### Without installing dependencies

Only skip `npm ci` when the Fallow config does not enable `typeAware`. Without `node_modules`, type-aware analysis degrades and reports code that is in use as unused.

```yaml
jobs:
  fallow:
    uses: digicatapult/shared-workflows/.github/workflows/fallow-npm.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
    with:
      install_dependencies: false
```

### Running locally

Use the same version as the workflow's `fallow_version`:

```bash
npx fallow@3.31.0
```
