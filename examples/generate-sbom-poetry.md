# Generating SBOMs (Poetry)

## Using [generate-sbom-poetry.yml](../.github/workflows/generate-sbom-poetry.yml) in callers

This workflow intentionally does **not** run `poetry install`. SBOM generation is performed from the Poetry project definition (and typically the `poetry.lock`), which avoids needing access to private package indexes and makes the SBOM reflect the lockfile.

If a repository does not commit `poetry.lock`, or needs lockfile generation as part of CI, you may want to add a repository-specific step to run `poetry lock` (or `poetry install`) before invoking this shared workflow.

If you want the shared workflow itself to install dependencies (closer to the NPM SBOM workflow behaviour), enable `poetry_install`:

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-poetry.yml@main
    permissions:
      contents: read
    with:
      poetry_install: true
      poetry_install_args: "--no-interaction --no-ansi --no-root"
```

### Minimal

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-poetry.yml@main
    permissions:
      contents: read
```

### Minimal using Dependency Track

To invoke Dependency Track upload, provide `DTRACK_APIKEY` and `DTRACK_HOSTNAME` and ensure that `inputs.enable_check_version` and `inputs.enable_dtrack_project` both resolve to `true`.

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-poetry.yml@main
    permissions:
      contents: read
    with:
      enable_check_version: true
      enable_dtrack_project: true
    secrets:
      DTRACK_APIKEY: ${{ secrets.DTRACK_APIKEY }}
      DTRACK_HOSTNAME: ${{ secrets.DTRACK_HOSTNAME }}
```
