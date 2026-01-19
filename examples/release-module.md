# Releasing packages

## Using [release-module.yml](../.github/workflows/release-module.yml) in callers

Several permissions are included in this workflow:
- `contents: write`
- `id-token: write`
- `packages: write`

They're invoked at the workflow level for the `publish-npm` job.

> [!TIP]
> Permission blocks in the caller workflow can be omitted and made implicit, to reduce maintenance when making changes to the callee. They can also be explicit, to minimise warnings from CodeQL scans and to enforce compliance. Any divergence in the permissions invoked can result in a workflow breaking, specifically where the caller assumes permissions that the callee isn't expecting. It's a trade-off between DRY principles, convenience, and compliance.


### Implicit permissions with defaults

A minimal workflow will publish an NPM package to a package registry.

```yaml
jobs:
  release-module:
    uses: digicatapult/shared-workflows/.github/workflows/release-module.yml@main
```


### Explicit permissions

```yaml
jobs:
  release-module:
    uses: digicatapult/shared-workflows/.github/workflows/release-module.yml@main
    permissions:
      contents: write
      id-token: write
      packages: write
```


### Minimal using third-party registries

Using external registries, such as [npmjs.org](https://npmjs.org), requires the `REGISTRY_AUTH_TOKEN` secret. Certain registries (NPMJS) can be configured to accept an OCID token to confirm the provenance of a given release, and that typically becomes visible in the entry for that version with a green tick in the registry. Third-party assertations of provenance using OCID tokens is the aspect of the job that requires the `id-token: write` permission.

```yaml
jobs:
  release-module:
    uses: digicatapult/shared-workflows/.github/workflows/release-module.yml@main
    secrets:
      REGISTRY_AUTH_TOKEN: REGISTRY_AUTH_TOKEN
```
