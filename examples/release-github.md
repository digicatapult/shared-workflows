# Releasing on GitHub

## Using [release-github.yml](../.github/workflows/release-github.yml) in callers

Several permissions are included in this workflow:

- `pull-requests: read`
- `contents: write`

They're invoked at the workflow level for the `release` job. Writing contents ensures that the workflow can create new assets for the release. Reading pull requests is required to build the release notes from the most recently merged PR. Add `actions: read` when attaching release assets via `release_assets_artifact_name`.

### Explicit permissions

A minimal workflow will create a release on GitHub with tags for the given version and separately for `latest`.

```yaml
jobs:
  release-github:
    uses: digicatapult/shared-workflows/.github/workflows/release-github.yml@main
    permissions:
      pull-requests: read
      contents: write
```

### Minimal with dependencies

This kind of job in `release.yml` should be dependent on a build step succeededing, e.g. [build-docker.yml](../.github/workflows/build-docker.yml) via `needs: [build-docker]`, to help avoid broken releases.

```yaml
jobs:
  release-github:
    uses: digicatapult/shared-workflows/.github/workflows/release-github.yml@main
    needs: [build-docker]
    permissions:
      pull-requests: read
      contents: write
```

### Release assets

`release-github.yml` doesn't pick release assets itself. [stage-release-assets.yml](../.github/workflows/stage-release-assets.yml) collects SBOMs (`get_sbom`) and any other artefacts (`additional_release_artifact_patterns`), gives them unique names, writes `checksums.sha256` over them, and uploads all of it as one bundle artefact (`release-assets` by default). `release-github.yml` then downloads that bundle, checks every file against `checksums.sha256`, and attaches the files and the manifest.

```yaml
jobs:
  sbom:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    needs: [build-docker]
    permissions:
      contents: read

  stage-release-assets:
    uses: digicatapult/shared-workflows/.github/workflows/stage-release-assets.yml@main
    needs: [sbom]
    permissions:
      actions: read
    with:
      get_sbom: true

  release-github:
    uses: digicatapult/shared-workflows/.github/workflows/release-github.yml@main
    needs: [stage-release-assets]
    permissions:
      pull-requests: read
      contents: write
      actions: read
    with:
      release_assets_artifact_name: ${{ needs.stage-release-assets.outputs.artifact_name }}
```

### Multiple images and additional artefacts

For a release with multiple images, call `build-docker` and `generate-sbom` once per image with distinct image, Dockerfile and SBOM names, and set `expected_sbom_count` to the number of SBOMs the release must contain. Other artefacts (e.g. Trivy container reports, ZAP reports) are picked with `additional_release_artifact_patterns`, an allowlist glob passed directly to `actions/download-artifact`'s `pattern` input. Use the extglob brace syntax (e.g. `{a,b}`) to match more than one artefact name. These files are renamed `<artefact-name>-<filename>`, so artefacts with the same internal filename (e.g. each image's Trivy report) don't collide; any collision that remains fails the staging job.

```yaml
jobs:
  build-api:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
    with:
      image_name: api
      docker_file: Dockerfile.api
      push_ghcr: true
      scan_container: true

  build-worker:
    # ...as above, with image_name: worker and docker_file: Dockerfile.worker

  sbom-api:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    needs: [build-api]
    permissions:
      contents: read
    with:
      sbom_output_file: api.cdx.json

  sbom-worker:
    # ...as above, with sbom_output_file: worker.cdx.json

  scan-zap:
    uses: digicatapult/shared-workflows/.github/workflows/scan-zap.yml@main
    needs: [build-api]
    permissions:
      contents: read
    with:
      docker_name: "ghcr.io/zaproxy/zaproxy:2.17.0"
      docker_compose_file: docker-compose.yml
      target: "http://localhost:3000"

  stage-release-assets:
    uses: digicatapult/shared-workflows/.github/workflows/stage-release-assets.yml@main
    needs: [build-api, build-worker, sbom-api, sbom-worker, scan-zap]
    permissions:
      actions: read
    with:
      get_sbom: true
      expected_sbom_count: 2
      additional_release_artifact_patterns: "{*-trivy-container-report,zap_scan-*}"

  release-github:
    uses: digicatapult/shared-workflows/.github/workflows/release-github.yml@main
    needs: [stage-release-assets]
    permissions:
      pull-requests: read
      contents: write
      actions: read
    with:
      release_assets_artifact_name: ${{ needs.stage-release-assets.outputs.artifact_name }}
```

### Attest, then release

`checksums.sha256` on its own is just a list of hashes. It sits in the same editable release as the files it lists, so it can't prove those files haven't been swapped. Signing it fixes that, and it should be signed **before** anything is published. Attest the staged bundle's manifest with [attest.yml](../.github/workflows/attest.yml), and make `release-github` depend on that job. If attestation fails, nothing is released. If the bundle changes after attestation, the `sha256sum --check` in `release-github.yml` fails. Either way, what ships is exactly what was signed.

```yaml
jobs:
  stage-release-assets:
    uses: digicatapult/shared-workflows/.github/workflows/stage-release-assets.yml@main
    needs: [build-docker, sbom]
    permissions:
      actions: read
    with:
      get_sbom: true
      additional_release_artifact_patterns: "*-trivy-container-report"

  attest-checksums:
    needs: [stage-release-assets]
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      subject-checksums-artifact-name: ${{ needs.stage-release-assets.outputs.artifact_name }}

  release-github:
    uses: digicatapult/shared-workflows/.github/workflows/release-github.yml@main
    needs: [stage-release-assets, attest-checksums]
    permissions:
      pull-requests: read
      contents: write
      actions: read
    with:
      release_assets_artifact_name: ${{ needs.stage-release-assets.outputs.artifact_name }}
```

To check a downloaded release asset, run `sha256sum --check --ignore-missing checksums.sha256` against the manifest, then `gh attestation verify <asset> --repo <owner>/<repo> --signer-repo digicatapult/shared-workflows --source-ref refs/heads/main` against the asset itself (see [Attest](../README.md#attest-examples) for why `--signer-repo` is needed).
