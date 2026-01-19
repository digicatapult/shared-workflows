# Building Docker images

## Using [build-docker.yml](../.github/workflows/build-docker.yml) in callers

<!--
Permissions should be at job level, as repo-ids doesn't use any.
contents: write needs to be verified; contents: read might be more accurate.
-->

Several permissions are included in this workflow:
- `contents: write`
- `packages: write`
- `security-events: write`

They're invoked at the workflow level for the `build-docker` job, but the `repo-ids` job doesn't require them.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

A minimal workflow will build an image without pushing it to a container registry. This is useful for testing that the image builds successfully.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
```


### Explicit permissions

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: write
      packages: write
      security-events: write
```


### Minimal with a registry push

To push to either Docker Hub or the GHCR, one of the two corresponding inputs must be set. Both secrets, `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`, are required.

```yaml
jobs:
  build-docker:
    uses: digicatapult/shared-workflows/.github/workflows/build-docker.yml@main
    permissions:
      contents: write
      packages: write
      security-events: write
    with:
      push_dockerhub: true
      push_ghcr: true
    secrets:
      DOCKERHUB_USERNAME: DOCKERHUB_USERNAME
      DOCKERHUB_TOKEN: DOCKERHUB_TOKEN
```
