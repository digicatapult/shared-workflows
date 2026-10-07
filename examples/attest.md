# Attesting workflow artifacts and images

## Using [attest.yml](../.github/workflows/attest.yml) in callers

`attest.yml` is a thin, generic wrapper around [`actions/attest`](https://github.com/actions/attest). It is deliberately **not** called from inside `build-docker.yml`, `generate-sbom.yml`, `scan-zap.yml`, or `release-github.yml`.

There are three typical use cases:
1. [Attesting a single artifact](#attesting-a-single-artifact), taking the name of the artifact
2. [Attesting a release's checksums manifest](#attesting-a-releases-checksums-manifest), taking the name of the SHA256 checksum file
3. [Attesting images](#attesting-images), taking both the common GHCR image name(s) and the associated predicates (artifacts)

Reusable workflow permission checks apply to every job defined in a called workflow file, regardless of whether an `if:` condition would skip that job at runtime. A job requesting `id-token: write` (or any other elevated permission) fails every caller that hasn't granted it, even when the step requesting it is conditional on an input the caller never sets. Embedding attestation inside the existing build/scan/release workflows would therefore break every existing caller of those workflows, whether or not they use attestation.

### Attesting a single artifact

The simplest case: attest a file produced by a previous job, as a plain SLSA build provenance attestation.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make my-app
      - uses: actions/upload-artifact@v7
        with:
          name: my-app
          path: my-app

  attest:
    needs: build
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      subject-artifact-name: my-app
```

`actions: read` is required because `attest.yml` downloads the named workflow artifact from the same run before attesting it.

### Attesting a release's checksums manifest

Attesting `subject-checksums` covers every file listed in a `sha256sum`-format manifest with a single attestation - this is the recommended way to attest exactly what a GitHub release shipped (see [release-github.yml examples](release-github.md#attesting-the-releases-checksums-manifest) for a full, working example that downloads `checksums.sha256` back from the release itself).

```yaml
jobs:
  attest-checksums:
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      subject-checksums: checksums.sha256
```

### Attesting images

To attest one or more images published by [build-docker.yml](build-docker.md), list them in `image-matrix`, and list the predicates to attach to each one in `predicate-matrix`. Both inputs are newline-separated strings (`|` block strings), because reusable workflow inputs cannot be YAML lists.

`attest.yml` makes one attestation per image and predicate pair, each in its own matrix job (e.g. `worker - trivy`). For each image it works out the rest:

| Value             | Mapping source                                                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Subject name      | `ghcr.io/<owner>/<image>`                                                                                                         |
| Subject digest    | the `<image>-image-digest` artifact, uploaded by `build-docker` when `push_ghcr: true`                                            |
| `trivy` predicate | the `<image>-trivy-container-report` artifact (`scan_container: true`), type `https://trivy.dev/report/v1`                        |
| `sbom` predicate  | the `<image>.cdx.json` artifact from `generate-sbom` (set `sbom_output_file: <image>.cdx.json`), type `https://cyclonedx.org/bom` |

If `predicate-matrix` is empty, each image gets a plain build provenance attestation rather than an error or warning.

With single-image repositories:
```yaml
  attest-image:
    needs: build-docker
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      image-matrix: ${{ github.event.repository.name }}
      predicate-matrix: trivy
```

With multi-image repositories and monorepos:
```yaml
jobs:
  build-worker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
    with:
      image_name: worker
      docker_file: worker/Dockerfile
      push_ghcr: true
      scan_container: true

  build-monolith:
    # ...as above, with image_name: monolith

  sbom-worker:
    uses: digicatapult/shared-workflows/.github/workflows/generate-sbom.yml@main
    permissions:
      contents: read
    with:
      sbom_output_file: worker.cdx.json

  sbom-monolith:
    # ...as above, with sbom_output_file: monolith.cdx.json

  attest-images:
    needs: [build-worker, build-monolith, sbom-worker, sbom-monolith]
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      image-matrix: |
        monolith
        worker
      predicate-matrix: |
        sbom
        trivy
```

The image names and predicate names are checked before anything is signed. Once signing starts, each attestation runs separately (`fail-fast: false`). If an artifact is missing, for example `trivy` is requested for an image built without `scan_container`, only that attestation fails.

> [!NOTE]
> The Trivy scan covers only the `linux/amd64` image, but the attestation is made against the multi-arch manifest-list digest. The `linux/arm64` image is **not** scanned, so a `trivy` attestation is limited in coverage to AMD64 only.

### Verifying an attestation

The signing identity is `digicatapult/shared-workflows`, not the calling repository, because `actions/attest` always signs as the workflow file that invokes it (not the workflow that called that workflow). The obvious `gh attestation verify <asset> --repo digicatapult/<repo>` is rejected as a result - you must add `--signer-repo`:

```bash
gh attestation verify <asset> \
  --repo digicatapult/<repo> \
  --signer-repo digicatapult/shared-workflows \
  --source-ref refs/heads/main
```

`--source-ref refs/heads/main` is recommended so that an attestation produced by a PR run - where caller-controlled inputs (e.g. a scanner image) could influence the predicate - isn't accepted as equivalent to a release-time attestation produced from `main`.
