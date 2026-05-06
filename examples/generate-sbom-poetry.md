# Generating SBOMs (Poetry)

## Using [generate-sbom.yml](../.github/workflows/generate-sbom.yml) in callers

By default, this workflow does **not** run `poetry install`. SBOM generation is performed from project files (typically `pyproject.toml` and `poetry.lock`).

For repositories that require dependency installation before SBOM generation (for example private indexes or generated lockfiles), you can opt in with `install_python_deps: true`.

For mixed repositories that contain both Node/NPM and Python/Poetry, the recommended way to constrain `@cyclonedx/cdxgen` to Python dependencies is to pass `--type python` via `additional_args`.

When `install_python_deps: true`:

- `python_install_command` can be provided for full control.
- If no custom command is provided, the workflow auto-detects install mode:
  - `package_manager: poetry` + `pyproject.toml` => runs `poetry install --no-interaction --no-ansi`
  - `requirements.txt` => runs `pip install -r requirements.txt`

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
      additional_args: "--type python"

### With Python dependency installation

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      package_manager: poetry
      sbom_tool: "@cyclonedx/cdxgen"
      additional_args: "--type python"
      install_python_deps: true
      python_version: "3.14"
```

### With custom Python install command

```yaml
jobs:
  generate-sbom-poetry:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      package_manager: poetry
      sbom_tool: "@cyclonedx/cdxgen"
      additional_args: "--type python"
      install_python_deps: true
      python_install_command: "pip install poetry && poetry install --no-interaction --no-ansi"
```
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
