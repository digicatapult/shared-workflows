# Generating SBOMs

## Using [generate-sbom-npm.yml](../.github/workflows/generate-sbom-npm.yml) in callers

> [!TIP]
> No extra job/workflow permissions are required by any of the options.


### Explicit permissions with defaults

A minimal workflow will create the SBOM against the working branch and upload it as an artifact, potentially for use by other jobs. This won't upload it to a Dependency Track host, however. It can be useful to check for issues with dependencies or the formatting of SBOMs, as the CycloneDX-NPM tool is quite strict.

```yaml
jobs:
  generate-sbom-npm:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-npm.yml@main
    permissions: {}
```


### Implicit permissions

```yaml
jobs:
  generate-sbom-npm:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-npm.yml@main
```


### Error handling

In practice, the CycloneDX tooling will fail on a range of errors, including excessive dependencies. To minimise the time spent debugging this, it's advisable to ignore the errors instead.

```yaml
jobs:
  generate-sbom-npm:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-npm.yml@main
    permissions: {}
    with:
      additional_args: "--ignore-npm-errors"
```


### Minimal using Dependency Track

To invoke the use of Dependency Track as a destination for the SBOMs, it's necessary to provide the secrets `DTRACK_APIKEY` and `DTRACK_HOSTNAME` and the ensure that `inputs.enable_check_version` and `inputs.enable_dtrack_project` both resolve to `true`.

```yaml
jobs:
  generate-sbom-npm:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom-npm.yml@main
    permissions: {}
    with:
      additional_args: "--ignore-npm-errors"
      enable_check_version: true
      enable_dtrack_project: true
    secrets:
      DTRACK_APIKEY: ${{ secrets.DTRACK_APIKEY }}
      DTRACK_HOSTNAME: ${{ secrets.DTRACK_HOSTNAME }}
```
