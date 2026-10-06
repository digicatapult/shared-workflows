# Fallow audit

## Using [fallow-npm.yml](../.github/workflows/fallow-npm.yml) in callers

Runs [Fallow](https://github.com/fallow-rs/fallow) on `pull_request` and reports unused code, duplication and complexity on the lines the PR changed, as a sticky PR comment and inline review comments. The audit is advisory by default and does not fail the job.

Fallow is installed by the workflow at `fallow_version`, so do not add it to the caller's `package.json`. Project-specific settings live in the caller's `.fallowrc.json`, which Fallow discovers automatically.

### Minimal

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
      fail_on_issues: false
```

### Blocking, without installing dependencies

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
      fail_on_issues: true
```

### Running locally

```bash
npx fallow@3.31.0
```
