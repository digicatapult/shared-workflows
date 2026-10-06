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

### With container CVE scanning

Setting `scan_container: true` runs an isolated, least-privilege Trivy scan of the built `linux/amd64` image and fails the build on `scan_fail_severity` (default `CRITICAL`). The report is uploaded as the `<image-name>-trivy-container-report` artifact. This requires no extra permissions on top of the standard `build-docker` set.

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
      scan_container: true
    secrets:
      DOCKERHUB_USERNAME: DOCKERHUB_USERNAME
      DOCKERHUB_TOKEN: DOCKERHUB_TOKEN
```

### Attesting the published image and its Trivy report

To get a signed, verifiable claim that *this* released image was scanned with *this* result, attest the image digest as the subject with the Trivy report as the predicate. This is done with a new job set after `build-docker` has pushed the image.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: read
      packages: write
      security-events: write
    with:
      push_dockerhub: false
      push_ghcr: true
      scan_container: true
    secrets:
      DOCKERHUB_USERNAME: DOCKERHUB_USERNAME
      DOCKERHUB_TOKEN: DOCKERHUB_TOKEN

  resolve-image-digest:
    needs: build-docker
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.inspect.outputs.digest }}
    steps:
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: inspect
        run: |
          DIGEST=$(docker buildx imagetools inspect "ghcr.io/${{ github.repository }}:${{ github.sha }}" --format '{{json .Manifest}}' | jq -r '.digest')
          echo "digest=${DIGEST}" >> "$GITHUB_OUTPUT"

  attest-image:
    needs: [build-docker, resolve-image-digest]
    permissions:
      id-token: write
      attestations: write
      artifact-metadata: write
      actions: read
    uses: digicatapult/shared-workflows/.github/workflows/attest.yml@main
    with:
      subject-name: ghcr.io/${{ github.repository }}
      subject-digest: ${{ needs.resolve-image-digest.outputs.digest }}
      predicate-type: https://trivy.dev/report/v1
      predicate-artifact-name: ${{ github.event.repository.name }}-trivy-container-report
```

Verify with `gh attestation verify <image-ref> --repo <owner>/<repo> --signer-repo digicatapult/shared-workflows --source-ref refs/heads/<branch>`.
