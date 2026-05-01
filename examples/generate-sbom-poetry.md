# Generating SBOMs (Poetry)

## Using [generate-sbom.yml](../.github/workflows/generate-sbom.yml) in callers

This workflow intentionally does **not** run `poetry install`. SBOM generation is performed from the Poetry project definition (and typically the `poetry.lock`), which avoids needing access to private package indexes and makes the SBOM reflect the lockfile.

If a repository does not commit `poetry.lock`, or needs lockfile generation as part of CI, you may want to add a repository-specific step to run `poetry lock` (or `poetry install`) before invoking this shared workflow.

If a repository needs dependency installation (e.g., private indexes, dynamic lockfile generation), add repository-specific steps before invoking this shared workflow.

### Minimal

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      package_manager: poetry
      sbom_tool: "@cyclonedx/cdxgen"
```

### Minimal using Dependency Track

To invoke Dependency Track upload, provide `DTRACK_APIKEY` and `DTRACK_HOSTNAME` and ensure that `inputs.enable_check_version` and `inputs.enable_dtrack_project` both resolve to `true`.

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      package_manager: poetry
      sbom_tool: "@cyclonedx/cdxgen"
      enable_check_version: true
      enable_dtrack_project: true
    secrets:
      DTRACK_APIKEY: ${{ secrets.DTRACK_APIKEY }}
      DTRACK_HOSTNAME: ${{ secrets.DTRACK_HOSTNAME }}
```
