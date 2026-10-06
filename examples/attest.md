# Attesting workflow artefacts and images

## Using [attest.yml](../.github/workflows/attest.yml) in callers

`attest.yml` is a thin, generic wrapper around [`actions/attest`](https://github.com/actions/attest). It is deliberately **not** called from inside `build-docker.yml`, `generate-sbom.yml`, `scan-zap.yml`, or `release-github.yml`.

Reusable workflow permission checks apply to every job defined in a called workflow file, regardless of whether an `if:` condition would skip that job at runtime. A job requesting `id-token: write` (or any other elevated permission) fails every caller that hasn't granted it, even when the step requesting it is conditional on an input the caller never sets. Embedding attestation inside the existing build/scan/release workflows would therefore break every existing caller of those workflows, whether or not they use attestation.

So: add a job for `attest.yml` directly in **your own** top-level workflow, after the job that produced the artefact or image, and grant the permissions on that job yourself. The permissions grant then becomes an explicit, visible action you take - not a surprise static failure for callers who never asked for it.

### Attesting a single file

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

### Attesting an image with a predicate

For a claim about a specific image - e.g. "this image was scanned with this Trivy result", rather than just "some run of this workflow produced some bytes" - attest the image digest as the subject with the scan report (or SBOM) as a custom predicate. See [build-docker.yml examples](build-docker.md#attesting-the-published-image-and-its-trivy-report) for a full, working example that resolves the image digest after `build-docker` has pushed it.

```yaml
jobs:
  attest-image:
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      subject-name: ghcr.io/org/image
      subject-digest: sha256:abc123...
      predicate-type: https://trivy.dev/report/v1
      predicate-artifact-name: trivy-container-report
```

### Verifying an attestation

The signing identity is `digicatapult/shared-workflows`, not the calling repository, because `actions/attest` always signs as the workflow file that invokes it (not the workflow that called that workflow). The obvious `gh attestation verify <asset> --repo digicatapult/<repo>` is rejected as a result - you must add `--signer-repo`:

```bash
gh attestation verify <asset> \
  --repo digicatapult/<repo> \
  --signer-repo digicatapult/shared-workflows \
  --source-ref refs/heads/main
```

`--source-ref refs/heads/main` is recommended so that an attestation produced by a PR run - where caller-controlled inputs (e.g. a scanner image) could influence the predicate - isn't accepted as equivalent to a release-time attestation produced from `main`.
