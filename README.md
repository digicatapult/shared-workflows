# shared-workflows

Shared github workflows created by the `digicatapult` organisation.

## Contents

**Version synchronisation** (bump on the pull request branch)

- [Synchronise PR Version](#synchronise-pr-version-examples)
- [Synchronise PR Version (Poetry)](#synchronise-pr-version-poetry-examples)
- [Synchronise all PR versions](#synchronise-all-pr-versions-examples)
- [Synchronise all PR versions (Poetry)](#synchronise-all-pr-versions-poetry-examples)

**Merge queue versioning** (bump on trunk after merge)

- [Require Version Label](#require-version-label-examples)
- [Apply Version](#apply-version-examples)
- [Apply Version (Poetry)](#apply-version-poetry-examples)

> These two groups are alternatives, not companions. A repository uses one or
> the other. See [Choosing a versioning model](#choosing-a-versioning-model).

**Build & release**

- [Build Docker](#build-docker-examples)
- [Check Version](#check-version-examples)
- [Release Github](#release-github-examples)
- [Release Module NPM](#release-module-npm-examples)

**Testing & quality**

- [Poetry Static checks](#poetry-static-checks-examples)
- [Poetry Tests](#poetry-tests-examples)
- [Poetry E2E tests](#poetry-e2e-tests-examples)
- [Poetry Migration Checks](#poetry-migration-checks-examples)
- [NPM Static Checks](#npm-static-checks-examples)
- [NPM E2E Tests](#npm-e2e-tests-examples)
- [NPM Tests](#npm-tests-examples)
- [NPM Migration Checks](#npm-migration-checks-examples)

**Security & analysis**

- [Generate SBOM](#generate-sbom-examples)
- [Generate Security Scorecard](#generate-security-scorecard-examples)
- [Scan Secrets](#scan-secrets-examples)
- [Scan Vulnerabilities](#scan-vulnerabilities-examples)
- [ZAP Scan](#zap-scan-examples)

## Choosing a versioning model

Two versioning models are supported. Both are maintained; pick one per repository and use its workflows exclusively, because running both would apply two bumps for a single change and both would compete over the same `v:` labels.

**Version synchronisation** is the default and is unchanged. Applying a `v:patch`, `v:minor` or `v:major` label makes the bot commit the resulting version onto the pull request branch, and `check-version` gates the release on that version being higher than the last release. It suits repositories that merge to trunk one pull request at a time.

**Merge queue versioning** suits repositories using a GitHub merge queue. The pull request carries only the label, stating the intent to bump, and the number is computed and committed on trunk after merge. Nothing is written to the pull request branch.

The distinction matters because a version written onto a branch is an absolute number. Two approved pull requests both compute the same next version, so the second one to reach the queue fails `check-version` and is evicted. Recovering it means bumping again, which dismisses its approval and removes it from the queue, which is the loop a merge queue is supposed to remove. Deferring the number to merge time makes the gate order independent, so any number of approved pull requests can flow through the queue together.

| | Version synchronisation | Merge queue versioning |
| --- | --- | --- |
| PR gate | `check-version` (version is higher than last release) | [Require Version Label](#require-version-label-examples) (exactly one `v:` label) |
| Where the number is written | The pull request branch, before merge | Trunk, after merge |
| Bump workflows | [Synchronise PR Version](#synchronise-pr-version-examples), [Synchronise all PR versions](#synchronise-all-pr-versions-examples) | [Apply Version](#apply-version-examples) |
| Concurrent approved PRs | Serialised, each re-bump dismisses approval | Flow through the queue together |
| GitHub App | Pushes to the pull request branch | Pushes to trunk, so it needs a ruleset bypass there |

Adopting the merge queue model also means callers must add a `merge_group` trigger to whichever workflow provides their required checks. A required check that never reports on the queue's branch leaves the queue waiting indefinitely.

### Merge queue compatibility

The `merge_group` event is neither a pull request nor a push, and the queue's branch (`gh-readonly-queue/<base>/pr-<n>-<sha>`) is ephemeral and deleted the moment the entry merges. Several workflows here needed adjusting for that. The behaviour is listed once here rather than repeated in each file.

| Workflow | Behaviour on `merge_group` | Why |
| --- | --- | --- |
| `migration-checks-npm` / `migration-checks-poetry` | Resolves base and head from `github.event.merge_group` | The payload has neither `pull_request` nor `before`. Without this the base sha resolves empty and the immutability lint, the ordering lint and seeded-upgrade all silently no-op for every queued pull request. |
| `tests-npm` / `tests-poetry` | Job name uses `current` / `main`, never the branch | Interpolating the branch puts the ephemeral queue ref into the check name, making it unique per pull request and per merge attempt. Such a name can never be listed as a required status check, so the queue would wait forever. |
| `tests-npm` | Coverage summary comment is skipped | There is no pull request to comment on. Thresholds are still enforced. |
| `static-checks-npm` / `static-checks-poetry` | GHAS SARIF upload is skipped | The queue deletes its ref before code scanning can attach the analysis, so the upload fails with "ref not found in this repository". The scan itself still runs, and the same commit is uploaded from its pull request run and again on push to trunk. |
| `require-version-label` | Performs no check, but still runs | The label was validated when the pull request was queued. The job runs rather than being skipped so the required check reports a result. |
| `build-docker` | Builds without pushing, and callers must set `fail_on_same_version: false` | Its `check-version` call defaults to failing when the version already matches the latest release. That is the pull request version gate for repos with no separate `check-version` job, but under merge queue versioning the branch deliberately carries no bump, so it would fail every pull request. `require-version-label` enforces the intent instead. Leave it at the default on `release.yml`, where the version commit is bumped. |

`release-github` is unaffected by `merge_group` since it runs on push, but its release notes are resolved from the commits since the previous release tag rather than from the most recently merged pull request, so a batched queue merge credits every pull request it contained.

## Workflows

> [!IMPORTANT]
> Shared workflow calls from private repositories require read permissions (e.g., `contents: read`, `actions: read`, `pull-requests: read`). Public repositories do not require these permissions.

The following workflows are included in this repository:

### [Synchronise PR Version](.github/workflows/synchronise-pr-version-npm.yml) ([examples](examples/synchronise-pr-version.md))

Synchronises the version in a `package.json` on a pull-request branch in relation to a trunk branch based on the presence of one of three labels: [`v:major`, `v:minor`, `v:patch`]. The workflow calculates the next version by incrementing the trunk branch version according to the label (major increments X.0.0, minor increments 0.X.0, patch increments 0.0.X). If the PR's current version doesn't match the expected version, the workflow commits the correction. The `v:stale` label is removed if present to indicate the version is now current. This workflow is typically called from repository-specific workflows when PR labels are added or changed.

#### Inputs

| Input        | Type   | Description                             | Default | Required |
| ------------ | ------ | --------------------------------------- | ------- | -------- |
| pr-number    | number | The PR to run this workflow for         |         | true     |
| trunk-branch | string | The trunk branch to synchronise against | `main`  | true     |

#### Permissions

| Access                 | Jobs used             | Level    | Reason                                                                            | Conditions |
| ---------------------- | --------------------- | -------- | --------------------------------------------------------------------------------- | ---------- |
| `contents: write`      | `synchronise-version` | Workflow | To POST commits against a pull request; required by `planetscale@ghcommit-action` | N/A        |
| `pull-requests: write` | `synchronise-version` | Workflow | To use `gh pr` to DELETE labels (`v:stale`) from affected PRs                     | N/A        |

#### Secrets

This workflow also requires two secrets in order to run:

| Secret    | Description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Synchronise PR Version (Poetry)](.github/workflows/synchronise-pr-version-poetry.yml) ([examples](examples/synchronise-pr-version-poetry.md))

Synchronises the version in a `pyproject.toml` on a pull-request branch in relation to a trunk branch based on the presence of one of three labels: [`v:major`, `v:minor`, `v:patch`]. Like the NPM variant, it calculates the next version by incrementing the trunk branch version and commits corrections as needed.

#### Inputs

| Input        | Type   | Description                             | Default | Required |
| ------------ | ------ | --------------------------------------- | ------- | -------- |
| pr-number    | number | The PR to run this workflow for         |         | true     |
| trunk-branch | string | The trunk branch to synchronise against | `main`  | true     |

#### Permissions

| Access                 | Jobs used             | Level    | Reason                                                                            | Conditions |
| ---------------------- | --------------------- | -------- | --------------------------------------------------------------------------------- | ---------- |
| `contents: write`      | `synchronise-version` | Workflow | To POST commits against a pull request; required by `planetscale@ghcommit-action` | N/A        |
| `pull-requests: write` | `synchronise-version` | Workflow | To use `gh pr` to DELETE labels (`v:stale`) from affected PRs                     | N/A        |

#### Secrets

This workflow also requires two secrets in order to run:

| Secret    | Description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Synchronise all PR versions](.github/workflows/synchronise-trunk-version-npm.yml) ([examples](examples/synchronise-trunk-version.md))

Synchronises the version in `package.json` for all open pull-requests that have one of the version labels: [`v:major`, `v:minor`, `v:patch`]. This workflow finds all PRs with these labels and calls the Synchronise PR Version workflow for each one. It's useful after the trunk branch version changes to ensure all open PRs have the correct calculated versions. Like the single PR workflow, it removes the `v:stale` label and commits corrections as needed.

#### Inputs

| Input        | Type   | Description                             | Default | Required |
| ------------ | ------ | --------------------------------------- | ------- | -------- |
| trunk-branch | string | The trunk branch to synchronise against | `main`  | true     |

#### Permissions

| Access                 | Jobs used                   | Level | Reason                                                                                     | Conditions |
| ---------------------- | --------------------------- | ----- | ------------------------------------------------------------------------------------------ | ---------- |
| `contents: read`       | `find-pull-requests`        | Job   | To GET repository contents                                                                 | N/A        |
| `pull-requests: read`  | `find-pull-requests`        | Job   | To GET open pull requests                                                                  | N/A        |
| `contents: write`      | `synchronise-pull-requests` | Job   | To invoke `synchronise-pr-version-npm.yml` and POST commits against all open pull requests | N/A        |
| `pull-requests: write` | `synchronise-pull-requests` | Job   | To use `gh pr` in the upstream workflow to DELETE labels (`v:stale`) from affected PRs     | N/A        |

#### Secrets

This workflow also requires two secrets in order to run:

| Secret    | Description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Synchronise all PR versions (Poetry)](.github/workflows/synchronise-trunk-version-poetry.yml) ([examples](examples/synchronise-trunk-version-poetry.md))

Synchronises the version in `pyproject.toml` for all open pull requests that have one of the version labels: [`v:major`, `v:minor`, `v:patch`]. This workflow finds all PRs with these labels and calls the Synchronise PR Version (Poetry) workflow for each one. It removes the `v:stale` label and commits corrections as needed.

#### Inputs

| Input        | Type   | Description                             | Default | Required |
| ------------ | ------ | --------------------------------------- | ------- | -------- |
| trunk-branch | string | The trunk branch to synchronise against | `main`  | true     |

#### Permissions

| Access                 | Jobs used                   | Level | Reason                                                                                        | Conditions |
| ---------------------- | --------------------------- | ----- | --------------------------------------------------------------------------------------------- | ---------- |
| `contents: read`       | `find-pull-requests`        | Job   | To GET repository contents                                                                    | N/A        |
| `pull-requests: read`  | `find-pull-requests`        | Job   | To GET open pull requests                                                                     | N/A        |
| `contents: write`      | `synchronise-pull-requests` | Job   | To invoke `synchronise-pr-version-poetry.yml` and POST commits against all open pull requests | N/A        |
| `pull-requests: write` | `synchronise-pull-requests` | Job   | To use `gh pr` in the upstream workflow to DELETE labels (`v:stale`) from affected PRs        | N/A        |

#### Secrets

This workflow also requires two secrets in order to run:

| Secret    | Description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Require Version Label](.github/workflows/require-version-label.yml) ([examples](examples/require-version-label.md))

The pull request version gate for repositories on a merge queue. Asserts that the pull request carries exactly one of `v:major`, `v:minor` or `v:patch`, failing with an actionable message when there is none or more than one. It replaces `check-version` as the pull request gate under [Merge queue versioning](#choosing-a-versioning-model): the label declares the intent to bump, and the number itself is applied after merge by [Apply Version](#apply-version-examples).

Labels are read from the event payload rather than the API, so no token and no permissions are needed.

#### Inputs

| Name             | Required | Type   | Default                               | Description                                                                     |
| ---------------- | -------- | ------ | ------------------------------------- | ------------------------------------------------------------------------------- |
| `version_labels` | No       | string | `'["v:major", "v:minor", "v:patch"]'` | JSON array of accepted labels. Exactly one must be present on the pull request. |

#### Permissions

None. Callers can pass `permissions: {}`.

#### Workflow Description

On a `pull_request` event the workflow compares the pull request's labels against `version_labels` and fails unless exactly one matches. On a `merge_group` event it performs no check and succeeds: the label was already validated when the pull request was queued, and the queue is merging that same pull request. The job still runs on `merge_group` rather than being skipped, so that the required check reports a result against the queue's branch. A required check that never reports leaves the merge queue waiting indefinitely.

Callers should give this workflow its own file triggered on the `labeled` and `unlabeled` pull request types as well as `merge_group`, so that adding a label re-evaluates the gate without re-running the repository's whole test suite.

### [Apply Version](.github/workflows/apply-version-npm.yml) ([examples](examples/apply-version.md))

Computes and commits the release version on trunk after a pull request has merged, for repositories using [Merge queue versioning](#choosing-a-versioning-model). It reads the version labels of every pull request contained in the push, takes the strongest one, increments trunk's `package.json` by that amount, and pushes a version commit back to trunk.

The increment arithmetic is identical to [Synchronise PR Version](#synchronise-pr-version-examples). The difference is where it runs: on trunk after merge rather than on the pull request branch before it, which is what stops two approved pull requests computing the same number.

#### Inputs

| Name                    | Required | Type   | Default           | Description                                                       |
| ----------------------- | -------- | ------ | ----------------- | ----------------------------------------------------------------- |
| `trunk-branch`          | No       | string | `main`            | Branch the version commit is pushed to.                           |
| `release-commit-prefix` | No       | string | `chore(release):` | Prefix of the version commit. Must match the caller's loop guard. |

#### Outputs

| Output    | Type    | Description                                                       |
| --------- | ------- | ----------------------------------------------------------------- |
| `bumped`  | boolean | `true` when a version commit was pushed.                          |
| `version` | string  | Version written to `package.json`, empty when nothing was bumped. |

#### Permissions

None. The workflow authenticates entirely with the GitHub App token, so callers can pass `permissions: {}`.

#### Secrets

| Name      | Required | Description             |
| --------- | -------- | ----------------------- |
| `bot-id`  | Yes      | GitHub App ID.          |
| `bot-key` | Yes      | GitHub App private key. |

#### Workflow Description

A merge queue can land several pull requests in a single push, so the workflow inspects every commit in the push rather than only the head commit, collecting the version labels of all associated pull requests and applying the strongest (major, then minor, then patch). A push containing no labelled pull request produces no bump and therefore no release.

The version commit is pushed with a GitHub App token rather than `GITHUB_TOKEN`. This is deliberate. `build-docker`, `generate-sbom` and `release-github` all read the version out of the checked out tree, so the commit that gets built and released must be the commit carrying the new version. Pushes made with `GITHUB_TOKEN` do not trigger workflows, so the version commit would never be built and `check-version` would report `is_new_version=false`. Using the App token means a merge produces two runs on trunk: the first applies the version, and the second, triggered by the version commit, releases it.

The `release-commit-prefix` is what keeps those two runs from triggering each other indefinitely. The caller guards its own invocation with `if: ${{ !startsWith(github.event.head_commit.message, '<prefix>') }}` and guards its release jobs with the inverse. The workflow repeats the check internally as a safety net.

This is the same GitHub App the `synchronise-*-version` workflows already use, so adopting repositories need no new app and no new secrets. The App does need to be added to the bypass list of the ruleset protecting trunk, because it now pushes there rather than to a feature branch.

### [Apply Version (Poetry)](.github/workflows/apply-version-poetry.yml) ([examples](examples/apply-version-poetry.md))

The Poetry counterpart of [Apply Version](#apply-version-examples). Behaviour is identical, reading and writing the version in `pyproject.toml` with `poetry version` instead of `package.json` with `npm version`. The version is read from `[tool.poetry]` and falls back to `[project]`, matching [Synchronise PR Version (Poetry)](#synchronise-pr-version-poetry-examples).

#### Inputs

| Name                    | Required | Type   | Default           | Description                                                       |
| ----------------------- | -------- | ------ | ----------------- | ----------------------------------------------------------------- |
| `trunk-branch`          | No       | string | `main`            | Branch the version commit is pushed to.                           |
| `python-version`        | No       | string | `3.14`            | Python version used to read `pyproject.toml` and run Poetry.      |
| `release-commit-prefix` | No       | string | `chore(release):` | Prefix of the version commit. Must match the caller's loop guard. |

#### Outputs

| Output    | Type    | Description                                                        |
| --------- | ------- | ------------------------------------------------------------------ |
| `bumped`  | boolean | `true` when a version commit was pushed.                           |
| `version` | string  | Version written to `pyproject.toml`, empty when nothing was bumped. |

#### Permissions

None. Callers can pass `permissions: {}`.

#### Secrets

| Name      | Required | Description             |
| --------- | -------- | ----------------------- |
| `bot-id`  | Yes      | GitHub App ID.          |
| `bot-key` | Yes      | GitHub App private key. |

#### Workflow Description

See [Apply Version](#apply-version-examples). The token model, the two-run release sequence and the commit prefix guard are the same.

### [Build Docker](.github/workflows/build-docker.yml) ([examples](examples/build-docker.md))

Builds a Docker container and optionally pushes it to GitHub Container Registry (GHCR), DockerHub, or both. The workflow is configurable to support multi-platform builds and various tagging options.

#### Inputs

| Input            | Type    | Description                                                                                                           | Default                     | Required |
| ---------------- | ------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------- | -------- |
| build_args       | string  | Build arguments to pass to Docker build                                                                               | `""`                        | false    |
| env_vars         | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run | `{}`                        | false    |
| pull_dhi         | boolean | Whether to login to Docker Hardened Images registry before building                                                   | `false`                     | false    |
| push_dockerhub   | boolean | Whether to push the built image to DockerHub                                                                          | `false`                     | false    |
| push_ghcr        | boolean | Whether to push the built image to GHCR                                                                               | `false`                     | false    |
| docker_platforms | string  | Specifies architectures to build the container for                                                                    | `"linux/amd64,linux/arm64"` | false    |
| docker_file      | string  | Dockerfile to be used for building the container                                                                      | `Dockerfile`                | false    |
| package_manager  | string  | Package manager for version detection, passed to `check-version`. Options: `npm`, `poetry`                            | `"npm"`                     | false    |
| arm64_runner     | string  | Runner label used for native `linux/arm64` builds. Must already exist as a GitHub-hosted runner in the org/repo       | `"ubuntu-24.04-arm"`        | false    |
| scan_container       | boolean | Scan the built `linux/amd64` image for CVEs with Trivy and fail on `scan_fail_severity`. Runs on any event (PR and release). | `false`                     | false    |
| trivy_image          | string  | Trivy scanner image, pinned by digest (renovate-updatable)                                                            | `aquasec/trivy:0.72.0@sha256:cffe3f…` | false |
| scan_fail_severity   | string  | Comma-separated severities that fail the build (the gate). The report artifact always contains every severity         | `CRITICAL`                  | false    |
| scan_ignore_unfixed  | boolean | When gating, ignore vulnerabilities with no fix available so the gate stays actionable (does not affect the report)   | `true`                      | false    |

Each platform in `docker_platforms` is built on its own native runner where one is known (`linux/amd64` → `ubuntu-latest`, `linux/arm64` → `arm64_runner`), falling back to `ubuntu-latest` with QEMU emulation for anything else. Per-platform images are pushed by digest and merged into a single multi-arch manifest, avoiding QEMU emulation for the common amd64/arm64 case.

When `scan_container` is enabled, the `build` job exports the built `linux/amd64` image as a tarball artifact and a separate `scan-image` job scans it with Trivy. It runs a single comprehensive scan: the uploaded `trivy-container-report` artifact (and the log) contain **every** severity, fixed and unfixed, so it is a usable inventory; the build then **fails** only on `scan_fail_severity` (default `CRITICAL`, computed from that same report). The scan job runs least-privilege (`contents: read`, **no secrets**) and Trivy is sandboxed to the image tarball, so a compromised scanner cannot reach the build job's registry credentials. Trivy is run from a **digest-pinned image**, deliberately not the `aquasecurity/trivy-action`, which was supply-chain compromised in March 2026 (see ENG-314). Results are an uploaded artifact only — there is no GHAS/SARIF upload for the Trivy scan (unlike the Docker Scout step, which still uploads on release). The manifest-tagging (`merge`) job depends on the scan, so a CRITICAL finding **prevents the consumable `:version`/`:latest` tag from being published** to GHCR/DockerHub (the per-arch layers are still pushed by digest by the `build` job, but remain untagged and are garbage-collected). When `scan_container` is `false` the tag is published as before.

#### Permissions

| Access                   | Jobs used             | Level | Reason                                                                    | Conditions                                 |
| ------------------------ | ---------------------- | ----- | -------------------------------------------------------------------------- | ------------------------------------------- |
| `contents: read`         | `prepare`               | Job   | To GET repository contents                                                | N/A                                         |
| `contents: read`         | `build`                 | Job   | To GET repository contents                                                | N/A                                         |
| `packages: write`        | `build`, `merge`        | Job   | To POST built packages/manifests to one or more container registries      | `inputs.push_dockerhub`/`inputs.push_ghcr`  |
| `security-events: write` | `merge`                 | Job   | To POST new code scanning alerts based on the SARIF report                | `inputs.push_dockerhub`/`inputs.push_ghcr`  |
| `contents: read`         | `scan-image`            | Job   | To download the exported image artifact and scan it (no secrets required) | `inputs.scan_container`                     |

#### Secrets

| Secret             | Description                                                                     |
| ------------------ | ------------------------------------------------------------------------------- |
| DOCKERHUB_USERNAME | DockerHub username used for authentication when pushing to DockerHub            |
| DOCKERHUB_TOKEN    | DockerHub token (or password) used for authentication when pushing to DockerHub |

#### Workflow Description

This GitHub Actions workflow is designed to build a Docker container, optionally pushing it to GHCR and/or DockerHub. It includes multi-platform support and customizable tagging for version management. Key steps are as follows:

1. **Repository Identifier Extraction**: Captures the repository and organization names, converting them to lowercase for consistency in tags.
2. **Set Environment Variables**: Parses JSON `env_vars` and sets them in the workflow environment.
3. **Version Management**: Uses `digicatapult/check-version` to determine the version and release type, which is then used in image tagging.
4. **Docker Setup**:
   - **QEMU Setup**: Enables emulation for multi-platform builds.
   - **Buildx Setup**: Sets up Docker Buildx for advanced build features.
5. **Tag Generation**: Based on version data and push preferences, generates tags for GHCR and DockerHub, including `:latest` tags for stable releases.
6. **Generate CVEs with Docker Scout**: Scout is used to generate a SARIF report from any CVEs found for the image.
7. **Upload Scout's SARIF results**: Any SARIF findings are pushed to GitHub's Advanced Security API.

#### Conditional Push Cases

1. **Push to GHCR Only**: If `push_ghcr` is true and `push_dockerhub` is false, builds and pushes the image to GHCR.
2. **Push to DockerHub Only**: If `push_dockerhub` is true and `push_ghcr` is false, builds and pushes the image to DockerHub.
3. **Push to Both GHCR and DockerHub**: If both `push_ghcr` and `push_dockerhub` are true, builds and pushes the image to both registries.
4. **Build Only**: If neither `push_ghcr` nor `push_dockerhub` are true, builds the image without pushing to any registry.

#### Labels and Metadata

For all images, the workflow adds OCI-compliant metadata labels to enrich the image with information about the repository source, version, and build date. This metadata provides valuable information for tracking the image’s origin and history.

This workflow is versatile, offering a full pipeline for Docker image creation and optional registry publishing, adapting to a range of requirements from simple builds to multi-platform, multi-registry deployments.

### [Check Version](.github/workflows/check-version.yml) ([examples](examples/check-version.md))

Determines the versioning information of the repository, setting output values related to version, release type, and build date. This workflow is typically used as a prerequisite step in other workflows that need version information for tagging, releasing, or publishing. It detects if the current commit represents a new version by comparing the `package.json` version against existing git tags, determines if the version follows pre-release naming conventions (e.g., contains alpha, beta, rc), and captures the build timestamp.

#### Outputs

| Output         | Type    | Description                                        |
| -------------- | ------- | -------------------------------------------------- |
| is_new_version | boolean | Indicates if the detected version is a new release |
| version        | string  | The current version string                         |
| build_date     | string  | The date when the build was initiated              |
| is_prerelease  | boolean | Indicates if the version is a pre-release          |

#### Permissions

| Access           | Jobs used       | Level    | Reason                                               | Conditions |
| ---------------- | --------------- | -------- | ---------------------------------------------------- | ---------- |
| `contents: read` | `check-version` | Workflow | To GET repository contents and version from git tags | N/A        |

#### Workflow Description

This GitHub Actions workflow verifies and checks the versioning details of the repository. It uses the `digicatapult/check-version` action to assess the version information, outputting details such as whether it’s a new version, the version string, if it's a pre-release, and the build date. These outputs can be used by subsequent jobs to conditionally perform tasks based on the versioning state.

### [Release Github](.github/workflows/release-github.yml) ([examples](examples/release-github.md))

Automates the release process on GitHub, creating a versioned release based on the repository's current version. This workflow is typically called after merging to the main branch when a new version is detected. It creates two GitHub releases: one with the specific version tag (e.g., v1.2.3) and another that updates the `latest` tag to point to this release. Release notes are automatically generated from the most recent merged PR's body, extracting sections for linked tickets, high-level description, and detailed description.

#### Inputs

| Input    | Type    | Description                                                                                                                               | Default | Required |
| -------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------- |
| env_vars | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`    | false    |
| get_sbom | boolean | An option to enable retrieval of SBOM artefacts from other workflows; leave `false` if none are expected                                  | `false` | false    |

#### Permissions

| Access                | Jobs used | Level    | Reason                                                                                        | Conditions        |
| --------------------- | --------- | -------- | --------------------------------------------------------------------------------------------- | ----------------- |
| `contents: write`     | `release` | Workflow | To POST release contents for both the versioned and `latest` tags                             | N/A               |
| `pull-requests: read` | `release` | Workflow | To GET an SBOM artifact from a PR, and then include it subsequently as a GitHub release asset | `inputs.get_sbom` |

#### Workflow Description

This GitHub Actions workflow creates a new release on GitHub. It uses the `digicatapult/check-version` action to determine the current version and then applies `softprops/action-gh-release` to create a versioned release and update the latest release tag. The process involves:

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Version Check**: Uses `digicatapult/check-version` to retrieve the current version information.
3. **Generate Release Notes**: Creates release notes based on the PR Body used by Digital Catapult.
4. **Build Versioned Release**: Creates a GitHub release using the version retrieved from the **Version Check** step and SBOMs if available.
5. **Build Latest Release**: Updates the `latest` tag to point to the newly created release.

This workflow helps streamline the release process by automating version checks and tagging, making it easy to manage versioned releases and update the latest release reference.

### [Release Module NPM](.github/workflows/release-module-npm.yml) ([examples](examples/release-module.md))

Publishes an NPM package to the specified registry, optionally building the package before publishing. This workflow supports configurable registry details, package access settings, and the option to build the package.

#### Inputs

| Input             | Type   | Description                                                                                                                               | Default                      | Required |
| ----------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | -------- |
| env_vars          | string | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                         | false    |
| node_version      | string | The node version to use                                                                                                                   | `24.x`                       | false    |
| registry_url      | string | The NPM registry URL to which the package will be published                                                                               | `https://registry.npmjs.org` | false    |
| registry_scope    | string | The NPM registry scope to use for the package                                                                                             | `@digicatapult`              | false    |
| npm_build_command | string | CLI command to run as a build step. Skipped if empty string                                                                               | `''`                         | false    |
| package_access    | string | The access level for the published package (`public` or `restricted`)                                                                     | `public`                     | false    |

#### Permissions

| Access            | Jobs used     | Level    | Reason                                                                                                       | Conditions |
| ----------------- | ------------- | -------- | ------------------------------------------------------------------------------------------------------------ | ---------- |
| `contents: write` | `publish-npm` | Workflow | To both GET and POST repository contents, as the READMEs are included in the package's documentation         | N/A        |
| `id-token: write` | `publish-npm` | Workflow | To request an OCID token and POST to confirm details about the provenance of a package in the NPMJS registry | N/A        |
| `packages: write` | `publish-npm` | Workflow | To POST a package to a GitHub Packages registry using `npm publish`                                          | N/A        |

#### Secrets

| Secret              | Description                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| REGISTRY_AUTH_TOKEN | Authentication token required to publish the package to the NPM registry. Only needed if `registry_url` is not `https://npm.pkg.github.com` |

#### Workflow Description

This GitHub Actions workflow publishes an NPM package, optionally building it beforehand. It is highly configurable, allowing users to specify registry details, scope, and access level. The workflow includes the following steps:

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Version Check**: Uses `digicatapult/check-version` to retrieve the current version information.
3. **Node Setup**: Configures the Node.js environment, including setting the registry URL and scope.
4. **Install Packages**: Installs the package dependencies.
5. **Build**: Builds the package if `npm_build_command` is provided.
6. **Publish to NPM**: Publishes the package to the specified registry with the given access level, using the provided authentication token.

This workflow simplifies the process of publishing NPM packages by handling environment setup, versioning, and publication in a single automated sequence.

### [Generate SBOM](.github/workflows/generate-sbom.yml) ([examples](examples/generate-sbom.md))

Generates a Software Bill of Materials (SBOM) using CycloneDX tools. This workflow supports both Node/NPM projects and Python/Poetry projects when `sbom_tool` is set to `@cyclonedx/cdxgen` (which supports `poetry.lock` / `pyproject.toml`) and `package_manager` is set to `poetry` for version detection. In mixed repositories, prefer passing `--type python` via `additional_args` when generating a Poetry-focused SBOM so the output is constrained to Python dependencies.

For backwards compatibility, the legacy filename [.github/workflows/generate-sbom-npm.yml](.github/workflows/generate-sbom-npm.yml) remains available as a thin wrapper around this workflow.

#### Inputs

| Input                  | Type    | Description                                                                                                                               | Default                               | Required |
| ---------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | -------- |
| dtrack_project_name    | string  | A project name to use within Dependency Track                                                                                             | `${{ github.event.repository.name }}` | false    |
| enable_check_version   | boolean | An option to enable the use of the digicatapult/check-version action                                                                      | `false`                               | false    |
| enable_dtrack_project  | boolean | An option to enable the use of Dependency Track                                                                                           | `false`                               | false    |
| package_manager        | string  | Package manager for version detection. Options: `npm`, `poetry`                                                                           | `npm`                                 | false    |
| env_vars               | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                                  | false    |
| node_version           | string  | The node version to use                                                                                                                   | `24.x`                                | false    |
| python_version         | string  | The Python version to use (for Poetry / `cdxgen` SBOMs)                                                                                   | `3.14`                                | false    |
| sbom_tool              | string  | SBOM generation tool to use. Options: `@cyclonedx/cyclonedx-npm`, `@cyclonedx/cdxgen`                                                     | `@cyclonedx/cyclonedx-npm`            | false    |
| sbom_format            | string  | SBOM output format. Options: `json`, `xml`                                                                                                | `json`                                | false    |
| sbom_output_file       | string  | Custom output filename. Defaults to `{repo-name}.cdx.{format}`                                                                            | `""`                                  | false    |
| npm_build_command      | string  | Optional build command to run before generating SBOM                                                                                      | `""`                                  | false    |
| additional_args        | string  | Additional arguments to pass to the SBOM generation tool                                                                                  | `""`                                  | false    |
| install_python_deps    | boolean | Install Python dependencies before SBOM generation                                                                                        | `false`                               | false    |
| python_install_command | string  | Optional custom command to install Python dependencies                                                                                    | `""`                                  | false    |
| upload_artifact        | boolean | Whether to upload the SBOM as a workflow artifact                                                                                         | `true`                                | false    |

#### Permissions

| Access           | Jobs used       | Level    | Reason                     | Conditions |
| ---------------- | --------------- | -------- | -------------------------- | ---------- |
| `contents: read` | `generate-sbom` | Workflow | To GET repository contents | N/A        |

#### Secrets

| Secret               | Description                                                                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `DTRACK_APIKEY`      | The Dependency Track API key; requires both the `BOM_UPLOAD` and `PROJECT_CREATION_UPLOAD` permissions                                          |
| `DTRACK_HOSTNAME`    | The hostname of the Dependency Track server; the HTTP schema is set separately with the inputs `protocol` (default: `https`) and `port` (`443`) |
| `DTRACK_PARENT_GUID` | The project GUID for the parent project within Dependency Track, typically named after the project repository and tagged with 'parent'          |

#### Outputs

| Output    | Type   | Description                         |
| --------- | ------ | ----------------------------------- |
| sbom_file | string | The name of the generated SBOM file |

#### Workflow Description

This GitHub Actions workflow generates an SBOM for a project. It allows flexibility in choosing the generation tool and format.

1. **Set Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Node Setup**: Configures the Node.js environment.
3. **Install Packages**: Installs project dependencies.
4. **Build (Optional)**: Runs a build command if specified, useful for projects that need compilation before SBOM generation.
5. **Generate SBOM**: Uses the selected tool (`@cyclonedx/cyclonedx-npm` or `@cyclonedx/cdxgen`) to generate the SBOM.
6. **Upload Artifact**: Optionally uploads the generated SBOM file as a workflow artifact.
7. **Upload SBOM to Dependency Track**: Optionally uploads the CycloneDX SBOM to a DT server. Docker Scout SBOMs are currently incompatible with DT due to inaccuracies in the CycloneDX spec implementation; CycloneDX-NPM is a more faithful implementation. To upload successfully, the step must have a DT hostname via the `DTRACK_HOSTNAME` secret, a parent project GUID (`DTRACK_PARENT_GUID`), and an API key (`DTRACK_APIKEY`) with both the `BOM_UPLOAD` and `PROJECT_CREATION_UPLOAD` permissions.

### [Generate Security Scorecard](.github/workflows/generate-security-scorecard.yml) ([examples](examples/generate-security-scorecard.md))

Runs an [OpenSSF Scorecard](https://github.com/ossf/scorecard) analysis against the callee repository. By default this runs the full `ossf/scorecard-action` check suite. When `custom_scan` is `true`, it instead runs the Scorecard CLI directly (via Docker), restricted to the checks listed in `custom_checks` (e.g. `Branch-Protection`), since `ossf/scorecard-action` has no input for selecting individual checks.

#### Inputs

| Input             | Type    | Description                                                                                                                                                                | Default         | Required |
| ----------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | -------- |
| results_file      | string  | File path to store the Scorecard results                                                                                                                                   | `results.sarif` | false    |
| results_format    | string  | Format of the results. Options: `sarif`, `json`                                                                                                                            | `sarif`         | false    |
| upload_type       | string  | How results are published. Options: `artifact`/`artefact` uploads a workflow artifact, `sarif` uploads to GitHub Code Scanning; unrecognised values are ignored            | `artefact`      | false    |
| publish_results   | boolean | Publishes results to the public [Scorecard REST API](https://api.scorecard.dev) and enables the Scorecard badge. Only supported by the default path - see the note below   | `false`         | false    |
| file_mode         | string  | Method used to fetch repository files. Options: `archive`, `git`                                                                                                           | `archive`       | false    |
| custom_scan       | boolean | Runs the Scorecard CLI directly (via Docker), restricted to `custom_checks`, instead of the full `ossf/scorecard-action` suite                                             | `false`         | false    |
| custom_checks     | string  | Comma-separated list of Scorecard check names to run when `custom_scan` is `true`, e.g. `Branch-Protection,Code-Review`. Required (non-empty) when `custom_scan` is `true` | `""`            | false    |
| scorecard_version | string  | Pinned tag of the `ghcr.io/ossf/scorecard` CLI image, used only when `custom_scan` is `true`                                                                               | `v5.5.0`        | false    |

#### Secrets

| Secret       | Description                                                                      | Required |
| ------------ | -------------------------------------------------------------------------------- | -------- |
| `repo_token` | GitHub token with read access, used to authenticate Scorecard's GitHub API calls | false    |

#### Permissions

| Access                    | Jobs used  | Level    | Reason                                                                                                                | Conditions                   |
| ------------------------- | ---------- | -------- | ----------------------------------------------------------------------------------------------------------------------| ---------------------------- |
| `security-events: write`  | `analysis` | Workflow | To POST new code scanning alerts based on the SARIF report                                                            | N/A                          |
| `id-token: write`         | `analysis` | Workflow | Required by `ossf/scorecard-action` to publish results/OIDC flows                                                     | N/A                          |
| `contents: read`          | `analysis` | Workflow | To check out the callee repository, and required by checks such as Branch-Protection, Code-Review and Signed-Releases | Required for private callers |
| `actions: read`           | `analysis` | Workflow | Required by Scorecard checks such as Token-Permissions to inspect workflow configuration                              | Required for private callers |
| `pull-requests: read`     | `analysis` | Workflow | Required by Scorecard checks such as Code-Review to evaluate pull request review history                              | Required for private callers |

#### Workflow Description

This GitHub Actions workflow runs an OpenSSF Scorecard analysis and publishes the results.

1. **Checkout code**: Checks out the callee repository.
2. **Validate custom scan inputs**: Fails fast with a clear error if `custom_scan` is `true` but `custom_checks` is empty, rather than letting the CLI silently run all checks.
3. **Validate upload/format combination**: Fails fast with a clear error if `results_format` is `json` and `upload_type` is `sarif`, since `github/codeql-action/upload-sarif` requires a SARIF file.
4. **Run analysis** _(default path, `custom_scan: false`)_: Runs `ossf/scorecard-action`, which always executes the complete set of Scorecard checks.
5. **Run custom analysis** _(`custom_scan: true`)_: Runs the pinned `ghcr.io/ossf/scorecard` CLI image directly via `docker run`, passing `--checks` for the `custom_checks` list, `--format`/`--file-mode` matching the other inputs, and `--output` to the shared `results_file` path so the upload steps below work unchanged. `ENABLE_SARIF=1` is set automatically when `results_format` is `sarif`, since SARIF output is feature-flagged in the CLI.
6. **Upload artifact** / **Upload to code-scanning**: Uploads `results_file` as a workflow artifact or to GitHub's Code Scanning dashboard, depending on `upload_type`.

#### `custom_scan` limitations

> [!NOTE]
> `publish_results` and the Scorecard badge/REST API integration are **not available** when `custom_scan` is `true`.
>
> `ossf/scorecard-action`'s `publish_results` input does two things that cannot be reimplemented with the Scorecard CLI:
>
> 1. It submits results for public repositories to the [Scorecard REST API](https://api.scorecard.dev), which OSSF ingests for cross-project analysis at scale.
> 2. It's a prerequisite for displaying a [Scorecard badge](https://openssf.org/blog/2022/09/08/show-off-your-security-score-announcing-scorecards-badges/) that gets updated in real time, thanks to the above REST API.
>
> Both integrations are tied to the full check suite produced by `ossf/scorecard-action`; a partial, custom-checks run isn't a valid input for either. If you need `publish_results`, use the default (`custom_scan: false`) path instead.

### [Poetry Static checks](.github/workflows/static-checks-poetry.yml) ([examples](examples/static-checks-poetry.md))

Runs static analysis for Poetry projects (default matrix includes `pylint`, `black`, `ruff`, `mypy`, and `bandit`). Each command runs as a separate matrix job so a single failing check does not interrupt the others. Optional TruffleHog (secrets) and Semgrep CE (vulnerabilities) scans can be enabled alongside the static analysis matrix.

#### Inputs

| Input                    | Type    | Description                                                                                                                               | Default                                                                           | Required |
| ------------------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | -------- |
| enable_semgrep_action    | boolean | An option to enable a Semgrep CE scan for bugs, security vulnerabilities, and compliance issues                                           | `true`                                                                            | false    |
| enable_trufflehog_action | boolean | An option to enable a TruffleHog GitHub Action, scanning for exposed secrets                                                              | `false`                                                                           | false    |
| env_vars                 | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                                                                              | false    |
| python_version           | string  | The Python version to use                                                                                                                 | `3.14`                                                                            | false    |
| working_directory        | string  | The working directory to run the checks from                                                                                              | `.`                                                                               | false    |
| matrix_commands          | string  | A JSON array of commands to run in the static checks matrix, each executed via `poetry run`                                               | `["pylint app", "black --check .", "ruff check .", "mypy app/", "bandit -r app"]` | false    |
| semgrep_extra_args       | string  | Extra arguments to be passed to the Semgrep CE CLI                                                                                        | `'--config="p/default"'`                                                          | false    |
| semgrep_sarif_path       | string  | A file path used to locate the SARIF result(s) from the Semgrep CLI                                                                       | `semgrep.sarif`                                                                   | false    |
| semgrep_upload_type      | string  | Upload format for Semgrep results; `sarif` uses the CodeQL SARIF upload action, `artefact` uses vanilla artefact upload, and `none` skips | `sarif`                                                                           | false    |
| trufflehog_extra_args    | string  | Extra arguments to be passed to the TruffleHog CLI                                                                                        | `--results=verified,unknown`                                                      | false    |

#### Permissions

| Access                   | Jobs used       | Level | Reason                                                     | Conditions                          |
| ------------------------ | --------------- | ----- | ---------------------------------------------------------- | ----------------------------------- |
| `contents: read`         | `static-checks` | Job   | To GET repository contents for static analysis             | N/A                                 |
| `contents: read`         | `scan-secrets`  | Job   | To GET repository contents and history for secret scanning | `inputs.enable_trufflehog_action`   |
| `contents: read`         | `scan-vulns`    | Job   | To GET repository contents for vulnerability scanning      | `inputs.enable_semgrep_action`      |
| `actions: read`          | `scan-vulns`    | Job   | To GET actions metadata for the scan                       | `inputs.enable_semgrep_action`      |
| `security-events: write` | `scan-vulns`    | Job   | To POST new code scanning alerts based on the SARIF report | `inputs.semgrep_upload_type: sarif` |

#### Workflow Description

This GitHub Actions workflow runs a configurable matrix of static checks on a Poetry project, optionally accompanied by secret and vulnerability scanning.

1. **Static Checks**: For each command in `matrix_commands`, sets up Python, installs Poetry and the project dependencies, then runs the command via `poetry run` in a dedicated matrix job (`fail-fast: false`).
2. **Secrets Scanning (Optional)**: When `enable_trufflehog_action` is `true`, runs TruffleHog against the branch for both verified and unverified secrets.
3. **Vulnerability Scanning (Optional)**: When `enable_semgrep_action` is `true` (and the actor is not `dependabot[bot]`), runs Semgrep CE and uploads the results either as a SARIF report to GitHub Advanced Security or as an artefact, depending on `semgrep_upload_type`.

### [Poetry Tests](.github/workflows/tests-poetry.yml) ([examples](examples/tests-poetry.md))

Runs unit/integration tests for Poetry projects using pytest and compares coverage between the PR branch and `main`. Each test path runs as a separate matrix job. When coverage is enabled, coverage is collected per test group on both the current branch and `main`, and a comparison summary is published to the job summary, failing the run if coverage drops below the configured thresholds.

#### Inputs

| Input               | Type    | Description                                                                                                                               | Default                               | Required |
| ------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | -------- |
| env_vars            | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                                  | false    |
| pre_test_command    | string  | Optional command to execute before the main test command                                                                                  | `""`                                  | false    |
| pull_ghcr           | boolean | Whether to login to GitHub Container Registry before docker compose                                                                       | `false`                               | false    |
| docker_compose_file | string  | The Docker Compose file to use for setting up dependencies                                                                                | `docker-compose.yml`                  | false    |
| python_version      | string  | The Python version to use                                                                                                                 | `3.14`                                | false    |
| working_directory   | string  | The working directory to run the tests from                                                                                               | `.`                                   | false    |
| tests               | string  | JSON array of test paths to run with pytest                                                                                               | `["tests/unit", "tests/integration"]` | false    |
| pytest_args         | string  | Extra arguments to pass to pytest                                                                                                         | `-v -s`                               | false    |
| coverage            | boolean | Whether to collect and report code coverage                                                                                               | `true`                                | false    |
| coverage_fail_under | number  | Fail the run if coverage for a test group falls below this percentage                                                                     | `50`                                  | false    |
| coverage_max_drop   | number  | Fail the run if coverage for a test group drops more than this many percentage points relative to `main`                                  | `0`                                   | false    |

#### Permissions

| Access           | Jobs used  | Level | Reason                                                                 | Conditions         |
| ---------------- | ---------- | ----- | ---------------------------------------------------------------------- | ------------------ |
| `contents: read` | `setup`    | Job   | To GET repository contents and determine branch information            | N/A                |
| `contents: read` | `tests`    | Job   | To GET repository contents and checkout different branches for testing | N/A                |
| `packages: read` | `tests`    | Job   | To GET packages from GitHub Container Registry when pulling images     | `inputs.pull_ghcr` |
| `contents: read` | `coverage` | Job   | To GET repository contents for coverage comparison                     | `inputs.coverage`  |

> [!NOTE]
> Unlike the [NPM Tests](#npm-tests-examples) workflow, the coverage comparison is written to the GitHub Actions job summary rather than posted as a pull-request comment, so `pull-requests: write` is not required.

#### Workflow Description

This GitHub Actions workflow runs pytest in a matrix strategy and, when enabled, compares coverage between the current branch and `main`.

1. **Setup**: Determines whether the run is on `main`, builds the branch matrix (current branch plus `main` for comparison when coverage is enabled), and generates a unique artifact prefix.
2. **Tests**: For each test path and branch, sets up Python, installs Poetry and dependencies, sets environment variables from `env_vars`, optionally logs in to GHCR and brings up Docker Compose dependencies, runs an optional `pre_test_command`, then runs pytest (under `coverage` when enabled) and uploads the coverage JSON per test group.
3. **Coverage**: Downloads the current-branch and `main` coverage summaries, builds a per-group comparison table in the job summary, and fails the run if any group is below `coverage_fail_under` or drops by more than `coverage_max_drop` relative to `main`.

### [Poetry E2E tests](.github/workflows/tests-e2e-poetry.yml) ([examples](examples/tests-e2e-poetry.md))

Runs end-to-end tests for Poetry projects (optionally using `docker-compose` / `docker bake`). Supports building dependency containers with Docker Bake, optionally setting up Node.js alongside Python, and running a customisable test command.

#### Inputs

| Input               | Type    | Description                                                                                                                               | Default                             | Required |
| ------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | -------- |
| env_vars            | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                                | false    |
| pre_test_command    | string  | Optional command to execute before the main test command                                                                                  | `""`                                | false    |
| pull_ghcr           | boolean | Whether to login to GitHub Container Registry before docker compose                                                                       | `false`                             | false    |
| docker_compose_file | string  | The Docker Compose file used for building the testing environment                                                                         | `docker-compose.yml`                | false    |
| python_version      | string  | The Python version to use                                                                                                                 | `3.14`                              | false    |
| setup_node          | boolean | Whether to also set up Node.js (e.g. for projects whose E2E tests use a Node-based runner)                                                | `false`                             | false    |
| node_version        | string  | The node version to use when `setup_node` is enabled                                                                                      | `24.x`                              | false    |
| working_directory   | string  | The working directory to run the tests from                                                                                               | `.`                                 | false    |
| test_command        | string  | Command used to run E2E tests, which can be customised as needed                                                                          | `poetry run pytest tests/e2e -v -s` | false    |

#### Permissions

| Access           | Jobs used   | Level | Reason                                                             | Conditions         |
| ---------------- | ----------- | ----- | ------------------------------------------------------------------ | ------------------ |
| `contents: read` | `e2e-tests` | Job   | To GET repository contents                                         | N/A                |
| `packages: read` | `e2e-tests` | Job   | To GET packages from GitHub Container Registry when pulling images | `inputs.pull_ghcr` |

#### Workflow Description

This GitHub Actions workflow runs end-to-end tests for a Poetry project within a Dockerised environment.

1. **Set Environment Variables**: Parses `env_vars` from JSON and applies them to the workflow environment.
2. **Container Setup (Optional)**: Optionally logs in to GHCR, then configures Docker Buildx and builds the E2E containers with Docker Bake from the provided Docker Compose file.
3. **Toolchain Setup**: Optionally sets up Node.js (when `setup_node` is `true`), then sets up Python and installs Poetry and the project dependencies.
4. **Pre-Test Command (Optional)**: Runs `pre_test_command` if provided.
5. **Run E2E Tests**: Executes `test_command`.

### [NPM Static Checks](.github/workflows/static-checks-npm.yml) ([examples](examples/static-checks.md))

Performs configurable static analysis checks on an NPM project, such as linting, dependency checking, XSS scanning, and additional checks, based on the specified commands.

#### Inputs

| Input                    | Type    | Description                                                                                                                               | Default                         | Required |
| ------------------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | -------- |
| enable_semgrep_action    | boolean | An option to enable a Semgrep CE scan for bugs, security vulnerabilities, and compliance issues                                           | `true`                          | false    |
| enable_trufflehog_action | boolean | An option to enable a TruffleHog GitHub Actions, scanning for exposed secrets                                                             | `false`                         | false    |
| env_vars                 | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                            | false    |
| node_version             | string  | The node version to use                                                                                                                   | `24.x`                          | false    |
| matrix_commands          | string  | A JSON array of commands to run in the static checks matrix, each representing an NPM script defined in the package                       | `["lint", "depcheck", "check"]` | false    |
| semgrep_extra_args       | string  | Extra arguments to be passed to the Semgrep CE CLI                                                                                        | `'--config="p/default"'`        | false    |
| semgrep_sarif_path       | string  | A file path used to locate the SARIF result(s) from the Semgrep CLI                                                                       | `semgrep.sarif`                 | false    |
| semgrep_upload_type      | string  | Upload format for Semgrep results; `sarif` uses the CodeQL SARIF upload action, `artefact` uses vanilla artefact upload, and `none` skips | `sarif`                         | false    |
| trufflehog_extra_args    | string  | Extra arguments to be passed to the TruffleHog CLI                                                                                        | `"--results=verified,unknown"`  | false    |

#### Permissions

| Access                   | Jobs used       | Level | Reason                                                     | Conditions                          |
| ------------------------ | --------------- | ----- | ---------------------------------------------------------- | ----------------------------------- |
| `contents: read`         | `scan-secrets`  | Job   | To GET repository contents and history for secret scanning | N/A                                 |
| `contents: read`         | `static-checks` | Job   | To GET repository contents for static analysis             | N/A                                 |
| `contents: read`         | `scan-vulns`    | Job   | To GET repository contents for vulnerability scanning      | N/A                                 |
| `actions: read`          | `scan-vulns`    | Job   | To GET actions metadata for the scan                       | N/A                                 |
| `security-events: write` | `scan-vulns`    | Job   | To POST new code scanning alerts based on the SARIF report | `inputs.semgrep_upload_type: sarif` |

#### Workflow Description

This GitHub Actions workflow runs a series of static checks on an NPM project based on a configurable list of commands. The commands run in parallel through a matrix strategy, allowing selective failure without interrupting the entire workflow.

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Node Setup**: Configures the Node.js environment.
3. **Node Modules Caching**: Caches `node_modules` based on the `package-lock.json` hash to speed up dependency installation.
4. **Install Packages**: Installs the necessary dependencies.
5. **Run Static Checks**: Executes each specified command in the matrix (`lint`, `depcheck`, `check` or others as configured) as defined in the NPM scripts.
6. **Secrets Scanning**: Run TruffleHog against the calling branch for both verified and unverified secrets.
7. **Vulnerability Scanning**: Run Semgrep CE to identify security vulnerabilities and upload the results in SARIF format to GitHub.

This flexible workflow enables dynamic static analysis checks to maintain code quality, making it adaptable to different project requirements.

### [NPM E2E Tests](.github/workflows/tests-e2e-npm.yml) ([examples](examples/tests-e2e.md))

Executes end-to-end (E2E) tests for an NPM project using Docker Compose, supporting optional build commands and project-specific configurations.

#### Inputs

| Input               | Type    | Description                                                                                                               | Default              | Required |
| ------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------- | -------- |
| env_vars            | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                 | false    |
| npm_build_command   | string  | Optional command to build the application before running tests                                                            | `""`                 | false    |
| pre_test_command    | string  | Optional command to execute before running the main test command                                                          | `""`                 | false    |
| pull_ghcr           | boolean | Whether to login to GitHub Container Registry before docker compose                                                       | `false`              | false    |
| docker_compose_file | string  | The Docker Compose file used for building the testing environment                                                         | `docker-compose.yml` | false    |
| node_version        | string  | The node version to use                                                                                                   | `24.x`               | false    |
| test_command        | string  | Command used to run E2E tests, which can be customized as needed                                                          | `"npm run test:e2e"` | false    |

#### Permissions

| Access           | Jobs used   | Level | Reason                                                             | Conditions         |
| ---------------- | ----------- | ----- | ------------------------------------------------------------------ | ------------------ |
| `contents: read` | `e2e-tests` | Job   | To GET repository contents and determine branch information        | N/A                |
| `packages: read` | `e2e-tests` | Job   | To GET packages from GitHub Container Registry when pulling images | `inputs.pull_ghcr` |

#### Workflow Description

This GitHub Actions workflow is tailored for running end-to-end tests within a Dockerized environment, with customizable build and test steps. The workflow supports conditional handling for specific repositories that require additional environment variables or secrets during test execution.

1. **Set Environment Variables**: Parses and applies environment variables from a JSON string to the workflow environment.
2. **Docker Buildx Setup**: Configures Docker Buildx for advanced Docker build support.
3. **Build E2E Containers**: Uses Docker Bake to build containers based on the provided Docker Compose file.
4. **Build Step (Optional)**: Runs the specified `npm_build_command` if provided.
5. **Pre-Test Command (Optional)**: Executes `pre_test_command` before running tests if specified.
6. **Run E2E Tests**: Executes the test command with repository-specific environment variables injected as needed.
7. **Upload Playwright Report**: Saves the Playwright report as an artifact, accessible for 90 days.
8. **Publish CTRF Test Summary**: Publishes a summary of the test results in CTRF format for comprehensive reporting.

This workflow is designed to accommodate different testing needs, offering flexibility with custom commands and environment-specific configurations for robust E2E testing.

### [NPM Tests](.github/workflows/tests-npm.yml) ([examples](examples/tests.md))

Runs specified NPM tests (e.g., unit and integration tests) with optional build and pre-test commands, as well as Docker-based dependency setup. Collects and reports code coverage by default, comparing coverage between the current branch and main branch.

#### Inputs

| Input                | Type    | Description                                                                                                           | Default                            | Required |
| -------------------- | ------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | -------- |
| env_vars             | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run | `{}`                               | false    |
| npm_build_command    | string  | Optional command to build the application before running tests                                                        | `""`                               | false    |
| pre_test_command     | string  | Optional command to execute before the main test command                                                              | `""`                               | false    |
| pull_ghcr            | boolean | Whether to login to GitHub Container Registry before docker compose                                                   | `false`                            | false    |
| docker_compose_file  | string  | The Docker Compose file to use for setting up dependencies                                                            | `docker-compose.yml`               | false    |
| node_version         | string  | The node version to use                                                                                               | `24.x`                             | false    |
| tests                | string  | JSON array of test commands defined in NPM scripts (e.g., `["test:unit", "test:integration"]`)                        | `["test:unit","test:integration"]` | false    |
| coverage             | boolean | Whether to collect and report code coverage                                                                           | `true`                             | false    |
| coverage_config_json | string  | Path to a custom c8 configuration JSON file                                                                           | `""`                               | false    |

#### Permissions

| Access                 | Jobs used  | Level | Reason                                                                      | Conditions         |
| ---------------------- | ---------- | ----- | --------------------------------------------------------------------------- | ------------------ |
| `contents: read`       | `setup`    | Job   | To GET repository contents and determine branch information                 | N/A                |
| `contents: read`       | `tests`    | Job   | To GET repository contents and checkout different branches for testing      | N/A                |
| `packages: read`       | `tests`    | Job   | To GET packages from GitHub Container Registry when pulling images          | `inputs.pull_ghcr` |
| `contents: read`       | `coverage` | Job   | To GET repository coverage config                                           | N/A                |
| `pull-requests: write` | `coverage` | Job   | To always POST coverage reports as comments for public/private repositories | `inputs.coverage`  |

#### Workflow Description

This GitHub Actions workflow runs a series of NPM test commands in a matrix strategy, testing both the current branch and main branch for coverage comparison. Each test command runs as a separate matrix job. When coverage is enabled, it collects coverage data from all test jobs and generates a comprehensive comparison report.

**Tests Job:**

1. **Set Environment Variables**: Parses `env_vars` from JSON and sets them in the workflow environment.
2. **Node Setup**: Installs the specified Node.js version.
3. **Node Modules Caching**: Caches `node_modules` based on `package-lock.json` for faster dependency installation.
4. **Install Packages**: Installs project dependencies using `npm ci`.
5. **Get Coverage Temp Directory**: Determines the coverage temporary directory from the provided config file or uses the default `coverage/tmp`.
6. **Build Step (Optional)**: Executes `npm_build_command` if it is provided.
7. **Environment File Creation**: Creates a `.env` file for environment configuration.
8. **Setup Docker Dependencies**: Brings up services defined in the specified Docker Compose file to support test dependencies.
9. **Wait for Initialization**: Adds a delay to ensure all Docker services are ready.
10. **Pre-Test Command (Optional)**: Runs `pre_test_command` if provided.
11. **Run Tests**: Executes each test command using c8 for coverage collection (e.g., `npx c8 npm run test:unit`).
12. **Upload Coverage**: If `coverage` is enabled, uploads coverage data from each test job, tagged by branch and test command.

**Coverage Job:**

1. **Setup c8 Config**: Uses the provided `coverage_config_json` if specified, otherwise creates a default configuration.
2. **Download Coverage Artifacts**: Downloads coverage summaries from both the current branch and main branch test jobs.
3. **Generate Reports**: Creates HTML, JSON summary, and JSON reports for the current branch coverage.
4. **Generate Main Branch Reports**: Creates JSON summary and JSON reports for the main branch coverage in a separate directory.
5. **Setup Vite Config**: Checks for an existing vite/vitest config file, creating a minimal one if none exists to prevent warnings from the coverage report action.
6. **Pretty Coverage Summary**: Uses `davelosert/vitest-coverage-report-action` to generate a formatted coverage report with trend indicators comparing current branch to main.
7. **Upload Reports for Current Branch**: Saves current branch coverage reports as an artifact.
8. **Fail if Under Thresholds**: Runs c8's threshold check, failing the workflow if coverage is below configured thresholds.

This workflow provides a comprehensive testing and coverage solution with branch comparison, custom build commands, Docker-based dependencies, and detailed coverage reporting including trend analysis.

### [NPM Migration Checks](.github/workflows/migration-checks-npm.yml) ([examples](examples/migration-checks.md))

Guards knex database migrations. Our other CI only ever migrates an empty database, and migrations run at container startup in production, so a migration that only fails against existing data (for example an `UPDATE` that violates a still-active `CHECK` constraint) is first exercised at deploy time. This workflow moves that failure into CI. It runs three jobs: a file lint (merged migrations are immutable, and new migrations must sort after existing ones), an up/down/up rollback roundtrip, and a seeded-upgrade job that migrates and seeds a database at the base commit, then applies only the new migrations on top. Postgres is started from the caller's own compose service by default so the CI version tracks what the application runs, with an image fallback.

Works on both `pull_request` and `push` callers. On `pull_request` the base and head are the PR base and head; on `push` (for example a `release.yml` on `main`) they are the commit before the push and the pushed commit, making seeded-upgrade a final pre-deploy "upgrade the previously deployed commit to this one" check. `migrate-roundtrip` runs on any event; the lint and seeded-upgrade jobs no-op when there is no base to compare against.

#### Inputs

| Input                 | Type   | Description                                                                                                | Default              | Required |
| --------------------- | ------ | --------------------------------------------------------------------------------------------------------- | -------------------- | -------- |
| node_version          | string | The node version to use                                                                                   | `24.x`               | false    |
| migrations_dir        | string | Path to the knex migrations directory, used by the file lint job                                          | `db/migrations`      | false    |
| migrate_command       | string | Command that applies all outstanding migrations                                                           | `npm run db:migrate` | false    |
| rollback_command      | string | Command that rolls back the most recent migration batch                                                   | `npm run db:rollback`| false    |
| seed_command          | string | Seed command for the seeded-upgrade job. Empty string skips seeding                                       | `""`                 | false    |
| postgres_compose_file | string | Compose file whose postgres service is started for the DB-backed jobs                                     | `docker-compose.yml` | false    |
| postgres_service      | string | Name of the postgres service within the compose file                                                     | `postgres`           | false    |
| postgres_image        | string | Fallback Postgres image; when set, a container from this image is used instead of the compose service     | `""`                 | false    |
| db_name               | string | Database to create when using the `postgres_image` fallback                                               | `postgres`           | false    |

#### Permissions

| Access           | Jobs used                                             | Level    | Reason                                                             | Conditions |
| ---------------- | ----------------------------------------------------- | -------- | ------------------------------------------------------------------ | ---------- |
| `contents: read` | `lint-migrations`, `migrate-roundtrip`, `seeded-upgrade` | Workflow | To GET repository contents and check out the base ref and PR head | N/A        |

### [Poetry Migration Checks](.github/workflows/migration-checks-poetry.yml) ([examples](examples/migration-checks-poetry.md))

The Alembic sibling of [NPM Migration Checks](#npm-migration-checks-examples), for Poetry projects. Same three jobs and the same rationale, with two mechanical differences: rollbacks use `alembic downgrade base`, and the file lint enforces that the revision graph has exactly one head (Alembic orders revisions by `down_revision`, not filename, so multiple heads are the equivalent of a knex "reorder" hazard) rather than a filename sort.

Works on both `pull_request` and `push` callers, the same way as the NPM variant. The single-head lint and `migrate-roundtrip` run on any event; the immutability lint and seeded-upgrade need a base commit to compare against.

#### Inputs

| Input                 | Type   | Description                                                                                            | Default                          | Required |
| --------------------- | ------ | ----------------------------------------------------------------------------------------------------- | -------------------------------- | -------- |
| python_version        | string | The python version to use                                                                             | `3.14`                           | false    |
| versions_dir          | string | Path to the alembic versions directory, used by the file lint job                                     | `alembic/versions`               | false    |
| migrate_command       | string | Command that applies all outstanding revisions                                                        | `poetry run alembic upgrade head`| false    |
| rollback_command      | string | Command that downgrades to the base (empty) revision                                                  | `poetry run alembic downgrade base`| false  |
| seed_command          | string | Seed command for the seeded-upgrade job. Empty string skips seeding                                   | `""`                             | false    |
| postgres_compose_file | string | Compose file whose postgres service is started for the DB-backed jobs                                 | `docker-compose.yml`             | false    |
| postgres_service      | string | Name of the postgres service within the compose file                                                 | `postgres`                       | false    |
| postgres_image        | string | Fallback Postgres image; when set, a container from this image is used instead of the compose service | `""`                             | false    |
| db_name               | string | Database to create when using the `postgres_image` fallback                                           | `postgres`                       | false    |

#### Permissions

| Access           | Jobs used                                             | Level    | Reason                                                             | Conditions |
| ---------------- | ----------------------------------------------------- | -------- | ------------------------------------------------------------------ | ---------- |
| `contents: read` | `lint-migrations`, `migrate-roundtrip`, `seeded-upgrade` | Workflow | To GET repository contents and check out the base ref and PR head | N/A        |

### [Scan Secrets](.github/workflows/scan-secrets.yml) ([examples](examples/scan-secrets.md))

Runs scanners to detect the exposure of secrets, with the option to add in extra arguments to exclude directories, fail on detection, or output the results in JSON format.

#### Inputs

| Input                    | Type    | Description                                                                   | Default                        | Required |
| ------------------------ | ------- | ----------------------------------------------------------------------------- | ------------------------------ | -------- |
| base                     | string  | An optional branch to base the scan on                                        | `""`                           | false    |
| enable_trufflehog_action | boolean | An option to enable a TruffleHog GitHub Actions, scanning for exposed secrets | `true`                         | false    |
| env_vars                 | string  | Extra variables to be passed to the environment                               | `{}`                           | false    |
| extra_args               | string  | Extra arguments to be passed to the TruffleHog CLI                            | `"--results=verified,unknown"` | false    |

#### Permissions

| Access           | Jobs used    | Level    | Reason                                                     | Conditions |
| ---------------- | ------------ | -------- | ---------------------------------------------------------- | ---------- |
| `contents: read` | `trufflehog` | Workflow | To GET repository contents and history for secret scanning | N/A        |

#### Workflow Description

This GitHub Actions workflow runs TruffleHog scans.

1. **Set Environment Variables**: Parses `env_vars` from JSON and sets them in the workflow environment.
2. **TruffleHog PR Scan**: Execute the TruffleHog scan of the PR, checking for verified and unverified secrets.

This workflow can be complemented with the complete list of TruffleHog arguments, found in that project's [repository](https://github.com/trufflesecurity/trufflehog/?tab=readme-ov-file#memo-usage).

### [Scan Vulnerabilities](.github/workflows/scan-vulns.yml) ([examples](examples/scan-vulns.md))

Runs scanners to detect bugs, security vulnerabilities, and compliance issues, with the option to add in extra arguments, e.g. additional Semgrep rulesets.

#### Inputs

| Input                 | Type    | Description                                                                                                                               | Default                  | Required |
| --------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ | -------- |
| enable_semgrep_action | boolean | An option to enable a Semgrep CE scan for bugs, security vulnerabilities, and compliance issues                                           | `true`                   | false    |
| extra_args            | string  | Extra arguments to be passed to the Semgrep CE CLI                                                                                        | `'--config="p/default"'` | false    |
| sarif_path            | string  | A file path used to locate the SARIF result(s) from the Semgrep CLI                                                                       | `semgrep.sarif`          | false    |
| upload_type           | string  | Upload format for Semgrep results; `sarif` uses the CodeQL SARIF upload action, `artefact` uses vanilla artefact upload, and `none` skips | `sarif`                  | false    |

#### Permissions

| Access                   | Jobs used | Level    | Reason                                                     | Conditions                  |
| ------------------------ | --------- | -------- | ---------------------------------------------------------- | --------------------------- |
| `security-events: write` | `semgrep` | Workflow | To POST new code scanning alerts based on the SARIF report | `inputs.upload_type: sarif` |
| `contents: read`         | `semgrep` | Workflow | To GET repository contents for vulnerability scanning      | N/A                         |
| `actions: read`          | `semgrep` | Workflow | To GET actions metadata for the scan                       | N/A                         |

#### Workflow Description

This GitHub Actions workflow runs Semgrep CE vulnerability scans.

1. **Semgrep CE Scan**: Execute the Semgrep CE CLI scan.
2. **Upload Semgrep CE's SARIF Results**: Upload the results to GitHub either as a SARIF report directly to the GitHub Advanced Security panel or as an artefact.

This workflow can be complemented with rulesets from the Semgrep [registry](https://semgrep.dev/r) of first- and third-party rules.

### [ZAP Scan](.github/workflows/scan-zap.yml) ([examples](examples/scan-zap.md))

Runs OWASP ZAP Dynamic Application Security Testing (DAST) scans against a running web application. Supports four scan types: `baseline`, `full`, `api`, and `automation-framework`, which are selectable via a JSON array matrix input. Multiple scan types can run in parallel. The workflow handles the initialisation of the application stack via a Docker Compose file and/or an arbitrary pre-scan command, then waits for the target URL to be reachable before handing off to ZAP.

#### Inputs

| Input               | Type    | Description                                                                                                                  | Default                            | Required |
| ------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | -------- |
| matrix_scans        | string  | JSON array of ZAP scan types to run: `baseline`, `full`, `api`, `automation-framework`                                       | `'["baseline"]'`                   | false    |
| target              | string  | URL of the web application to scan (used by `baseline`, `full`, and `api` scan types)                                        | `http://localhost:3000`            | false    |
| env_vars            | string  | JSON object of extra environment variables to inject                                                                         | `{}`                               | false    |
| docker_compose_file | string  | Docker Compose file used to bring up the application stack before scanning                                                   | `""`                               | false    |
| pull_ghcr           | boolean | Log in to GitHub Container Registry before pulling images (e.g. if using private/custom proxy image)                         | `false`                            | false    |
| pre_scan_command    | string  | Arbitrary shell command to start the application (e.g. `npm ci && npm start &`)                                              | `""`                               | false    |
| target_wait_timeout | number  | Seconds to poll `target` for a successful response before failing                                                            | `60`                               | false    |
| rules_file_name     | string  | Relative path to a ZAP rules TSV file for suppressing known alerts; applies to `baseline`, `full`, and `api` scan types only | `""`                               | false    |
| cmd_options         | string  | Additional ZAP CLI options passed to all scan types                                                                          | `""`                               | false    |
| docker_name         | string  | Custom ZAP Docker image name; defaults to the stable ZAP image                                                               | `"ghcr.io/zaproxy/zaproxy:stable"` | false    |
| allow_issue_writing | boolean | Create or update a GitHub issue in the repository with ZAP findings; requires `issues: write`                                | `false`                            | false    |
| fail_action         | boolean | Fail the job if ZAP identifies any alerts; applies to `baseline`, `full`, and `api` scan types only                          | `false`                            | false    |
| artifact_name       | string  | Base name for uploaded scan report artifacts; suffixed with the scan type (e.g. `zap_scan-baseline`)                         | `zap_scan`                         | false    |
| api_scan_format     | string  | API definition format for the `api` scan type: `openapi`, `soap`, or `graphql`                                               | `openapi`                          | false    |
| automation_plan     | string  | File path of the ZAP Automation Framework plan; required when using `automation-framework`                                   | `""`                               | false    |
| docker_env_vars     | string  | Newline-separated names of environment variables, used only by the `automation-framework` scan type                          | `""`                               | false    |

#### Permissions

| Access           | Jobs used  | Level    | Reason                                                                 | Conditions                   |
| ---------------- | ---------- | -------- | ---------------------------------------------------------------------- | ---------------------------- |
| `contents: read` | `zap-scan` | Workflow | To GET repository contents and pass rules files into the ZAP container | N/A                          |
| `packages: read` | `zap-scan` | Job      | To authenticate with GHCR and pull Docker images                       | `inputs.pull_ghcr`           |
| `issues: write`  | `zap-scan` | Job      | To allow ZAP to create or update a GitHub issue with scan findings     | `inputs.allow_issue_writing` |

#### Workflow Description

This GitHub Actions workflow runs OWASP ZAP DAST scans against a locally running web application.

1. **Set Environment Variables**: Parses `env_vars` from JSON and sets them in the workflow environment.
2. **Setup Dependencies** _(optional)_: Brings up the application using `docker compose up -d` against `docker_compose_file`. Skip this with `docker_compose_file: ''`.
3. **Run Pre-Scan Command** _(optional)_: Executes `pre_scan_command` on the runner for use cases where the application is started directly rather than via Docker Compose.
4. **Wait for Target** _(skipped for `automation-framework`)_: Polls `target` with `curl` at two-second intervals until it responds successfully or `target_wait_timeout` is reached.
5. **ZAP Scan**: Runs the selected ZAP action for each `matrix.scan_type`. Report artifacts are uploaded under `artifact_name-<scan_type>` to avoid collisions when multiple types run in parallel.
6. **Upload Automation Framework Report** _(only for `automation-framework`)_: Parses `automation_plan` for any `report` jobs, locates the generated report file(s) on the runner (via the `/zap/wrk` → `$GITHUB_WORKSPACE` mapping), and uploads them as an artifact under `artifact_name-automation-framework`.
