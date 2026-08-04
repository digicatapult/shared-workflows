# Requiring a version label

## Using [require-version-label.yml](../.github/workflows/require-version-label.yml) in callers

This is the pull request version gate for repositories on a **merge queue**. It asserts that a pull request carries exactly one of `v:major`, `v:minor` or `v:patch`.

It exists because the older gate does not survive a queue. Requiring that the version in `package.json` is higher than the last release puts an absolute number on the branch, so two approved pull requests necessarily compute the same number, and the second one to reach the queue fails `check-version` and is evicted. A label states the *intent* to bump instead, which is order independent, so every queued pull request can satisfy the gate at the same time. The number itself is applied on trunk after merge by [apply-version-npm.yml](../.github/workflows/apply-version-npm.yml) or [apply-version-poetry.yml](../.github/workflows/apply-version-poetry.yml).

No permissions are required. The labels are read from the event payload rather than the API, so the caller can pass `permissions: {}`.

### Give it its own workflow file

The gate needs to re-run when a label is added or removed, which means the `labeled` and `unlabeled` pull request types. Putting it in the repository's main `test.yml` would re-run the entire test suite every time anyone touches any label, so give it its own file.

The `merge_group` trigger is required. If a required check does not report on the queue's branch the merge queue waits indefinitely, so the workflow must run on `merge_group` even though there is nothing to validate there.

```yaml
name: Version label

on:
  pull_request:
    types: [opened, reopened, synchronize, labeled, unlabeled]
  merge_group:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  require-version-label:
    uses: digicatapult/shared-workflows/.github/workflows/require-version-label.yml@main
    permissions: {}
```

### Custom label set

A repository that uses different label names can override the accepted set. Exactly one of them must be present.

```yaml
jobs:
  require-version-label:
    uses: digicatapult/shared-workflows/.github/workflows/require-version-label.yml@main
    permissions: {}
    with:
      version_labels: '["release:major", "release:minor", "release:patch"]'
```

## Inputs

| Name             | Required | Type   | Default                                | Description                                                                 |
| ---------------- | -------- | ------ | -------------------------------------- | --------------------------------------------------------------------------- |
| `version_labels` | No       | string | `'["v:major", "v:minor", "v:patch"]'`  | JSON array of accepted labels. Exactly one must be present on the pull request. |

## Permissions

None. Pass `permissions: {}`.
