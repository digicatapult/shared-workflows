# shared-workflows

Shared github workflows created by the `digicatapult` organisation.

## Workflows

The following workflows are included in this repository

### [Synchronise PR Version](.github/workflows/synchronise-pr-version-npm.yml)

Synchronises the version in a `package.json` on a pull-request branch in relation to a trunk branch based on the presence of one of three labels: [`v:major`, `v:minor`, `v:patch`] and removes the label `v:stale` if present. If the current version is found to be incorrect this workflow will perform a commit on that branch in order to set the correct value.

#### Inputs

| input          | type     | description                             |
| -------------- | -------- | --------------------------------------- |
| `pr-number`    | `number` | The PR to run this workflow for         |
| `trunk-branch` | `string` | The trunk branch to synchronise against |

This workflow also requires two secrets in order to run

| secret    | description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Synchronise all PR versions](.github/workflows/synchronise-trunk-version-npm.yml)

Synchronises the version in a `package.json` in relation to a trunk branch based on the presence of one of three labels: [`v:major`, `v:minor`, `v:patch`] and removes the label `v:stale` if present. If the current version is found to be incorrect this workflow will perform a commit on that branch in order to set the correct value. This is done for all open pull-requests.

#### Inputs

| input          | type     | description                             |
| -------------- | -------- | --------------------------------------- |
| `trunk-branch` | `string` | The trunk branch to synchronise against |

This workflow also requires two secrets in order to run

| secret    | description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |
