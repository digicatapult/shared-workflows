# Generating SBOMs

## Using [generate-sbom.yml](../.github/workflows/generate-sbom.yml) in callers

> [!NOTE]
> The legacy workflow filename `generate-sbom-npm.yml` is still supported as a wrapper, but new callers should use `generate-sbom.yml`.

### Explicit permissions

A minimal workflow will create the SBOM against the working branch and upload it as an artifact, potentially for use by other jobs. This won't upload it to a Dependency Track host, however. It can be useful to check for issues with dependencies or the formatting of SBOMs, as the CycloneDX-NPM tool is quite strict.

```yaml
jobs:
  generate-sbom:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
```

### Error handling

In practice, the CycloneDX tooling will fail on a range of errors, including excessive dependencies. To minimise the time spent debugging this, it's advisable to ignore the errors instead.

```yaml
jobs:
  generate-sbom:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      additional_args: "--ignore-npm-errors"
```

### Minimal using Dependency Track

To invoke the use of Dependency Track as a destination for the SBOMs, it's necessary to provide the secrets `DTRACK_APIKEY` and `DTRACK_HOSTNAME` and the ensure that `inputs.enable_check_version` and `inputs.enable_dtrack_project` both resolve to `true`.

```yaml
jobs:
  generate-sbom:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      additional_args: "--ignore-npm-errors"
      enable_check_version: true
      enable_dtrack_project: true
    secrets:
      DTRACK_APIKEY: ${{ secrets.DTRACK_APIKEY }}
      DTRACK_HOSTNAME: ${{ secrets.DTRACK_HOSTNAME }}
      DTRACK_PARENT_GUID: ${{ secrets.DTRACK_PARENT_GUID }}

```

### Python/Poetry projects (via cdxgen)

`@cyclonedx/cdxgen` supports Poetry lockfiles (`poetry.lock` / `pyproject.toml`). This lets Python projects reuse the same SBOM workflow.

For mixed repositories that contain both Node/NPM and Python/Poetry, pass `--type python` via `additional_args` to ensure the generated SBOM only includes Python dependencies.

```yaml
jobs:
  generate-sbom-python:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      package_manager: poetry
      sbom_tool: "@cyclonedx/cdxgen"
      sbom_format: "json"
      additional_args: "--type python"
      enable_check_version: true
      enable_dtrack_project: true
    secrets:
      DTRACK_APIKEY: ${{ secrets.DTRACK_APIKEY }}
      DTRACK_HOSTNAME: ${{ secrets.DTRACK_HOSTNAME }}
      DTRACK_PARENT_GUID: ${{ secrets.DTRACK_PARENT_GUID }}
```
