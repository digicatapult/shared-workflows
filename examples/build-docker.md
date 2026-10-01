# Building Docker images

## Using [build-docker.yml](../.github/workflows/build-docker.yml) in callers

Several permissions are needed for the `build-docker` job:

- `contents: read`
- `packages: write`
- `security-events: write`

They should be invoked at the job level. Reading READMEs and LICENSE information and writing packages are both essential to the steps invoking `docker/build-push-action`. If `build-docker` is invoked without the option to push either to Docker Hub or GHCR, if merely testing the image, then in practice these levels of access aren't applied.

To upload details about any potential security vulnerabilities (CVEs) via GitHub's Code Scanning APIs, the ability to write security events is needed. These alerts are visible within the GitHub repository's Security panel. If GitHub Code Scanning is disabled at the time the workflow is executed, then the step will fail. These steps only run on public repositories.

### Explicit permissions

A very minimal workflow will build an image without pushing it to a container registry. This is useful for testing that the image builds successfully.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
```

### Minimal with a registry push

To push to either Docker Hub or the GHCR, one of the two corresponding inputs must be set. Both secrets, `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`, are required. This use case would suit a `release.yml` workflow, to push a successful build artifact (an image) to a registry.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
    with:
      push_dockerhub: true
      push_ghcr: true
    secrets:
      DOCKERHUB_USERNAME: DOCKERHUB_USERNAME
      DOCKERHUB_TOKEN: DOCKERHUB_TOKEN
```

### With container CVE scanning and a provenance attestation

Setting `scan_container: true` runs an isolated, least-privilege Trivy scan of the built `linux/amd64` image and fails the build on `scan_fail_severity` (default `CRITICAL`). Adding `attest_provenance: true` additionally generates a signed build provenance attestation (via `actions/attest`) for the Trivy report, so it can be attached to a GitHub release (see [release-github.yml](release-github.md)) and verified independently of the workflow's 30-day artifact retention. This requires the extra permissions below on top of the standard `build-docker` set.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
      id-token: write
      attestations: write
      artifact-metadata: write
    with:
      push_dockerhub: true
      push_ghcr: true
      scan_container: true
      attest_provenance: true
    secrets:
      DOCKERHUB_USERNAME: DOCKERHUB_USERNAME
      DOCKERHUB_TOKEN: DOCKERHUB_TOKEN
```
