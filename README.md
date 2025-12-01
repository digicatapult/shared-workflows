# shared-workflows

Shared github workflows created by the `digicatapult` organisation.

## Workflows

The following workflows are included in this repository

### [Synchronise PR Version](.github/workflows/synchronise-pr-version-npm.yml)

Synchronises the version in a `package.json` on a pull-request branch in relation to a trunk branch based on the presence of one of three labels: [`v:major`, `v:minor`, `v:patch`]. The workflow calculates the next version by incrementing the trunk branch version according to the label (major increments X.0.0, minor increments 0.X.0, patch increments 0.0.X). If the PR's current version doesn't match the expected version, the workflow commits the correction. The `v:stale` label is removed if present to indicate the version is now current. This workflow is typically called from repository-specific workflows when PR labels are added or changed.

#### Inputs

| input          | type     | description                             | default |
| -------------- | -------- | --------------------------------------- | ------- |
| `pr-number`    | `number` | The PR to run this workflow for         |         |
| `trunk-branch` | `string` | The trunk branch to synchronise against | `main`  |

This workflow also requires two secrets in order to run

| secret    | description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Synchronise all PR versions](.github/workflows/synchronise-trunk-version-npm.yml)

Synchronises the version in `package.json` for all open pull-requests that have one of the version labels: [`v:major`, `v:minor`, `v:patch`]. This workflow finds all PRs with these labels and calls the Synchronise PR Version workflow for each one. It's useful after the trunk branch version changes to ensure all open PRs have the correct calculated versions. Like the single PR workflow, it removes the `v:stale` label and commits corrections as needed.

#### Inputs

| input          | type     | description                             | default |
| -------------- | -------- | --------------------------------------- | ------- |
| `trunk-branch` | `string` | The trunk branch to synchronise against | `main`  |

This workflow also requires two secrets in order to run

| secret    | description                                                   |
| --------- | ------------------------------------------------------------- |
| `bot-id`  | Id of the `Github App` to use when committing version updates |
| `bot-key` | Private Key for the `Github App`                              |

### [Build Docker](.github/workflows/build-docker.yml)

Builds a Docker container and optionally pushes it to GitHub Container Registry (GHCR), DockerHub, or both. The workflow is configurable to support multi-platform builds and various tagging options.

#### Inputs

| Input            | Type    | Description                                                                                                           | Default                     | Required |
| ---------------- | ------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------- | -------- |
| build_args       | string  | Build arguments to pass to Docker build                                                                               | `""`                        | false    |
| env_vars         | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run | `{}`                        | false    |
| push_dockerhub   | boolean | Whether to push the built image to DockerHub                                                                          | `false`                     | false    |
| push_ghcr        | boolean | Whether to push the built image to GHCR                                                                               | `false`                     | false    |
| docker_platforms | string  | Specifies architectures to build the container for                                                                    | `"linux/amd64,linux/arm64"` | false    |
| docker_file      | string  | Dockerfile to be used for building the container                                                                      | `Dockerfile`                | false    |

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
6. **Generate SBOMs with Docker Scout**: If set to push to a registry, then Scout is used to generate an SBOM and a SARIF report from the image.
7. **Upload SBOM**: An SBOM is included with the PR as an artefact and also uploaded as a release asset based on the versioned tag used for the image.
8. **Upload Scout's SARIF results**: Depending on the destination selected, the SARIF results are pushed as an artefact or via CodeQL to GHAS.

#### Conditional Push Cases

1. **Push to GHCR Only**: If `push_ghcr` is true and `push_dockerhub` is false, builds and pushes the image to GHCR.
2. **Push to DockerHub Only**: If `push_dockerhub` is true and `push_ghcr` is false, builds and pushes the image to DockerHub.
3. **Push to Both GHCR and DockerHub**: If both `push_ghcr` and `push_dockerhub` are true, builds and pushes the image to both registries.
4. **Build Only**: If neither `push_ghcr` nor `push_dockerhub` are true, builds the image without pushing to any registry.

#### Labels and Metadata

For all images, the workflow adds OCI-compliant metadata labels to enrich the image with information about the repository source, version, and build date. This metadata provides valuable information for tracking the image’s origin and history.

This workflow is versatile, offering a full pipeline for Docker image creation and optional registry publishing, adapting to a range of requirements from simple builds to multi-platform, multi-registry deployments.

### [Check Version](.github/workflows/check-version.yml)

Determines the versioning information of the repository, setting output values related to version, release type, and build date. This workflow is typically used as a prerequisite step in other workflows that need version information for tagging, releasing, or publishing. It detects if the current commit represents a new version by comparing the `package.json` version against existing git tags, determines if the version follows pre-release naming conventions (e.g., contains alpha, beta, rc), and captures the build timestamp.

#### Outputs

| Output         | Type    | Description                                        |
| -------------- | ------- | -------------------------------------------------- |
| is_new_version | boolean | Indicates if the detected version is a new release |
| version        | string  | The current version string                         |
| build_date     | string  | The date when the build was initiated              |
| is_prerelease  | boolean | Indicates if the version is a pre-release          |

#### Workflow Description

This GitHub Actions workflow verifies and checks the versioning details of the repository. It uses the `digicatapult/check-version` action to assess the version information, outputting details such as whether it’s a new version, the version string, if it's a pre-release, and the build date. These outputs can be used by subsequent jobs to conditionally perform tasks based on the versioning state.

### [Release Github](.github/workflows/release-github.yml)

Automates the release process on GitHub, creating a versioned release based on the repository's current version. This workflow is typically called after merging to the main branch when a new version is detected. It creates two GitHub releases: one with the specific version tag (e.g., v1.2.3) and another that updates the `latest` tag to point to this release. Release notes are automatically generated from the most recent merged PR's body, extracting sections for linked tickets, high-level description, and detailed description.

#### Inputs

| Input    | Type    | Description                                                                                                                               | Default |
| -------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| env_vars | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`    |
| get_sbom | boolean | An option to disable the retrieval of SBOM artefacts, e.g. if none are expected from other workflows                                      | true    |

#### Workflow Description

This GitHub Actions workflow creates a new release on GitHub. It uses the `digicatapult/check-version` action to determine the current version and then applies `softprops/action-gh-release` to create a versioned release and update the latest release tag. The process involves:

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Version Check**: Uses `digicatapult/check-version` to retrieve the current version information.
3. **Generate Release Notes**: Creates release notes based on the PR Body used by Digital Catapult.
4. **Build Versioned Release**: Creates a GitHub release using the version retrieved from the **Version Check** step.
5. **Build Latest Release**: Updates the `latest` tag to point to the newly created release.

This workflow helps streamline the release process by automating version checks and tagging, making it easy to manage versioned releases and update the latest release reference.

### [Release Module NPM](.github/workflows/release-module-npm.yml)

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

### [NPM Static Checks](.github/workflows/static-checks-npm.yml)

Performs configurable static analysis checks on an NPM project, such as linting, dependency checking, XSS scanning, and additional checks, based on the specified commands.

#### Inputs

| Input                    | Type   | Description                                                                                                                               | Default                        | Required |
| ------------------------ | ------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | -------- |
| enable_semgrep_action    | bool   | An option to enable a Semgrep CE scan for bugs, security vulnerabilities, and compliance issues                                           | true                           | false    |
| enable_trufflehog_action | bool   | An option to enable a TruffleHog GitHub Actions, scanning for exposed secrets                                                             | false                          | false    |
| env_vars                 | string | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                           | false    |
| node_version             | string | The node version to use                                                                                                                   | `24.x`                         | false    |
| matrix_commands          | string | A JSON array of commands to run in the static checks matrix, each representing an NPM script defined in the package                       | `["lint","depcheck","check"]`  | false    |
| semgrep_extra_args       | string | Extra arguments to be passed to the Semgrep CE CLI                                                                                        | `'--config="p/default"'`       | false    |
| semgrep_sarif_path       | string | A file path used to locate the SARIF result(s) from the Semgrep CLI                                                                       | `semgrep.sarif`                | false    |
| semgrep_upload_type      | string | Upload format for Semgrep results; "sarif" uses the CodeQL SARIF upload action, "artefact" uses vanilla artefact upload, and "none" skips | `sarif`                        | false    |
| trufflehog_extra_args    | string | Extra arguments to be passed to the TruffleHog CLI                                                                                        | `"--results=verified,unknown"` | true     |

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

### [NPM E2E Tests](.github/workflows/e2e-tests-npm.yml)

Executes end-to-end (E2E) tests for an NPM project using Docker Compose, supporting optional build commands and project-specific configurations.

#### Inputs

| Input               | Type   | Description                                                                                                               | Default              | Required |
| ------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------- | -------------------- | -------- |
| env_vars            | string | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                 | false    |
| npm_build_command   | string | Optional command to build the application before running tests                                                            | `""`                 | false    |
| pre_test_command    | string | Optional command to execute before running the main test command                                                          | `""`                 | false    |
| docker_compose_file | string | The Docker Compose file used for building the testing environment                                                         | `docker-compose.yml` | false    |
| node_version        | string | The node version to use                                                                                                   | `24.x`               | false    |
| test_command        | string | Command used to run E2E tests, which can be customized as needed                                                          | `"npm run test:e2e"` | false    |

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

### [NPM Tests](.github/workflows/tests-npm.yml)

Runs specified NPM tests (e.g., unit and integration tests) with optional build and pre-test commands, as well as Docker-based dependency setup. Collects and reports code coverage by default, comparing coverage between the current branch and main branch.

#### Inputs

| Input                | Type    | Description                                                                                                           | Default                                     | Required |
| -------------------- | ------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | -------- |
| env_vars             | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run | `{}`                                        | false    |
| npm_build_command    | string  | Optional command to build the application before running tests                                                        | `""`                                        | false    |
| pre_test_command     | string  | Optional command to execute before the main test command                                                              | `""`                                        | false    |
| docker_compose_file  | string  | The Docker Compose file to use for setting up dependencies                                                            | `docker-compose.yml`                        | false    |
| node_version         | string  | The node version to use                                                                                               | `24.x`                                      | false    |
| tests                | string  | JSON array of test commands defined in NPM scripts (e.g., `["test:unit", "test:integration"]`)                        | `["test:unit","test:integration"]`          | false    |
| coverage             | boolean | Whether to collect and report code coverage                                                                           | `true`                                      | false    |
| coverage_config_json | string  |  Path to a custom c8 configuration JSON file                                                                          | `""`                                        | false    |


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

### [Scan Secrets](.github/workflows/scan-secrets.yml)

Runs scanners to detect the exposure of secrets, with the option to add in extra arguments to exclude directories, fail on detection, or output the results in JSON format.

#### Inputs

| Input                    | Type   | Description                                                                   | Default                        | Required |
| ------------------------ | ------ | ----------------------------------------------------------------------------- | ------------------------------ | -------- |
| base                     | string | An optional branch to base the scan on                                        | `""`                           | false    |
| enable_trufflehog_action | bool   | An option to enable a TruffleHog GitHub Actions, scanning for exposed secrets | false                          | true     |
| env_vars                 | string | Extra variables to be passed to the environment                               | `{}`                           | false    |
| extra_args               | string | Extra arguments to be passed to the TruffleHog CLI                            | `"--results=verified,unknown"` | true     |

#### Workflow Description

This GitHub Actions workflow runs TruffleHog scans.

1. **Set Environment Variables**: Parses `env_vars` from JSON and sets them in the workflow environment.
2. **TruffleHog PR Scan**: Execute the TruffleHog scan of the PR, checking for verified and unverified secrets.

This workflow can be complemented with the complete list of TruffleHog arguments, found in that project's [repository](https://github.com/trufflesecurity/trufflehog/?tab=readme-ov-file#memo-usage).

### [Scan Vulnerabilities](.github/workflows/scan-vulns.yml)

Runs scanners to detect bugs, security vulnerabilities, and compliance issues, with the option to add in extra arguments, e.g. additional Semgrep rulesets.

#### Inputs

| Input                 | Type   | Description                                                                                                                               | Default                  | Required |
| --------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------- | -------------------------| -------- |
| enable_semgrep_action | bool   | An option to enable a Semgrep CE scan for bugs, security vulnerabilities, and compliance issues                                           | true                     | false    |
| extra_args            | string | Extra arguments to be passed to the Semgrep CE CLI                                                                                        | `'--config="p/default"'` | false    |
| sarif_path            | string | A file path used to locate the SARIF result(s) from the Semgrep CLI                                                                       | `semgrep.sarif`          | false    |
| upload_type           | string | Upload format for Semgrep results; "sarif" uses the CodeQL SARIF upload action, "artefact" uses vanilla artefact upload, and "none" skips | `sarif`                  | false    |

#### Workflow Description

This GitHub Actions workflow runs Semgrep CE vulnerability scans.

1. **Semgrep CE Scan**: Execute the Semgrep CE CLI scan.
2. **Upload Semgrep CE's SARIF Results**: Upload the results to GitHub either as a SARIF report directly to the GitHub Advanced Security panel or as an artefact.

This workflow can be complemented with rulesets from the Semgrep [registry](https://semgrep.dev/r) of first- and third-party rules.
