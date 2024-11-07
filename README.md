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

### [Build Docker](.github/workflows/build-docker.yml)

Builds a Docker container and optionally pushes it to GitHub Container Registry (GHCR), DockerHub, or both. The workflow is configurable to support multi-platform builds and various tagging options.

#### Inputs

| Input            | Type    | Description                                                                                                             | Default                     | Required |
|------------------|---------|-------------------------------------------------------------------------------------------------------------------------|-----------------------------|----------|
| env_vars         | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run  | `{}`                        | false    |
| push_dockerhub   | boolean | Whether to push the built image to DockerHub                                                                            | `false`                     | false    |
| push_ghcr        | boolean | Whether to push the built image to GHCR                                                                                 | `false`                     | false    |
| docker_platforms | string  | Specifies architectures to build the container for                                                                      | `"linux/amd64,linux/arm64"` | false    |
| docker_file      | string  | Dockerfile to be used for building the container                                                                        | `Dockerfile`                | false    |

#### Secrets

| Secret             | Description                                                                                     |
|--------------------|-------------------------------------------------------------------------------------------------|
| DOCKERHUB_USERNAME | DockerHub username used for authentication when pushing to DockerHub                            |
| DOCKERHUB_TOKEN    | DockerHub token (or password) used for authentication when pushing to DockerHub                 |

#### Workflow Description

This GitHub Actions workflow is designed to build a Docker container, optionally pushing it to GHCR and/or DockerHub. It includes multi-platform support and customizable tagging for version management. Key steps are as follows:

1. **Repository Identifier Extraction**: Captures the repository and organization names, converting them to lowercase for consistency in tags.
2. **Set Environment Variables**: Parses JSON `env_vars` and sets them in the workflow environment.
3. **Version Management**: Uses `digicatapult/check-version` to determine the version and release type, which is then used in image tagging.
4. **Docker Setup**:
   - **QEMU Setup**: Enables emulation for multi-platform builds.
   - **Buildx Setup**: Sets up Docker Buildx for advanced build features.
5. **Tag Generation**: Based on version data and push preferences, generates tags for GHCR and DockerHub, including `:latest` tags for stable releases.

#### Conditional Push Cases

1. **Push to GHCR Only**: If `push_ghcr` is true and `push_dockerhub` is false, builds and pushes the image to GHCR.
2. **Push to DockerHub Only**: If `push_dockerhub` is true and `push_ghcr` is false, builds and pushes the image to DockerHub.
3. **Push to Both GHCR and DockerHub**: If both `push_ghcr` and `push_dockerhub` are true, builds and pushes the image to both registries.
4. **Build Only**: If neither `push_ghcr` nor `push_dockerhub` are true, builds the image without pushing to any registry.

#### Labels and Metadata

For all images, the workflow adds OCI-compliant metadata labels to enrich the image with information about the repository source, version, and build date. This metadata provides valuable information for tracking the image’s origin and history.

This workflow is versatile, offering a full pipeline for Docker image creation and optional registry publishing, adapting to a range of requirements from simple builds to multi-platform, multi-registry deployments.


### [Check Version](.github/workflows/check-version.yml)

Determines the versioning information of the repository, setting output values related to version, release type, and build date.

#### Outputs

| Output          | Type    | Description                                                               |
|-----------------|---------|---------------------------------------------------------------------------|
| is_new_version  | boolean | Indicates if the detected version is a new release                        |
| version         | string  | The current version string                                                |
| build_date      | string  | The date when the build was initiated                                     |
| is_prerelease   | boolean | Indicates if the version is a pre-release                                 |

#### Workflow Description

This GitHub Actions workflow verifies and checks the versioning details of the repository. It uses the `digicatapult/check-version` action to assess the version information, outputting details such as whether it’s a new version, the version string, if it's a pre-release, and the build date. These outputs can be used by subsequent jobs to conditionally perform tasks based on the versioning state.

### [Release Github](.github/workflows/release-github.yml)

Automates the release process on GitHub, creating a versioned release based on the repository’s current version.

#### Inputs

| Input      | Type    | Description                                                                                                             | Default |
|------------|---------|-------------------------------------------------------------------------------------------------------------------------|---------|
| env_vars   | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`    |

#### Workflow Description

This GitHub Actions workflow creates a new release on GitHub. It uses the `digicatapult/check-version` action to determine the current version and then applies `marvinpinto/action-automatic-releases` to create a versioned release and update the latest release tag. The process involves:

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Version Check**: Uses `digicatapult/check-version` to retrieve the current version information.
3. **Build Versioned Release**: Creates a GitHub release using the version retrieved from the previous step.
4. **Build Latest Release**: Updates the `latest` tag to point to the newly created release.

This workflow helps streamline the release process by automating version checks and tagging, making it easy to manage versioned releases and update the latest release reference.

### [Release Module NPM](.github/workflows/release-module-npm.yml)

Publishes an NPM package to the specified registry, optionally building the package before publishing. This workflow supports configurable registry details, package access settings, and the option to build the package.

#### Inputs

| Input            | Type    | Description                                                                                                             | Default                       | Required |
|------------------|---------|-------------------------------------------------------------------------------------------------------------------------|-------------------------------|----------|
| env_vars         | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                          | false    |
| registry_url     | string  | The NPM registry URL to which the package will be published                                                             | `https://registry.npmjs.org`  | false    |
| registry_scope   | string  | The NPM registry scope to use for the package                                                                           | `@digicatapult`               | false    |
| npm_build        | boolean | Whether to run a build step before publishing                                                                           | `false`                       | false    |
| package_access   | string  | The access level for the published package (`public` or `restricted`)                                                  | `public`                      | false    |

#### Secrets

| Secret       | Description                                                                   |
|--------------|-------------------------------------------------------------------------------|
| REGISTRY_AUTH_TOKEN   | Authentication token required to publish the package to the NPM registry      |

#### Workflow Description

This GitHub Actions workflow publishes an NPM package, optionally building it beforehand. It is highly configurable, allowing users to specify registry details, scope, and access level. The workflow includes the following steps:

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Version Check**: Uses `digicatapult/check-version` to retrieve the current version information.
3. **Node Setup**: Configures the Node.js environment, including setting the registry URL and scope.
4. **Install Packages**: Installs the package dependencies.
5. **Build**: Builds the package if `npm_build` is set to true.
6. **Publish to NPM**: Publishes the package to the specified registry with the given access level, using the provided `AUTH_TOKEN` for authentication.

This workflow simplifies the process of publishing NPM packages by handling environment setup, versioning, and publication in a single automated sequence.

### [NPM Static Checks](.github/workflows/static-checks-npm.yml)

Performs configurable static analysis checks on an NPM project, such as linting, dependency checking, XSS scanning, and additional checks, based on the specified commands.

#### Inputs

| Input           | Type    | Description                                                                                                             | Default                         | Required |
|-----------------|---------|-------------------------------------------------------------------------------------------------------------------------|---------------------------------|----------|
| env_vars        | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                            | false    |
| matrix_commands | string  | A JSON array of commands to run in the static checks matrix, each representing an NPM script defined in the package     | `[lint,depcheck,check]` | false    |

#### Workflow Description

This GitHub Actions workflow runs a series of static checks on an NPM project based on a configurable list of commands. The commands run in parallel through a matrix strategy, allowing selective failure without interrupting the entire workflow.

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Node Setup**: Configures the Node.js environment.
3. **Node Modules Caching**: Caches `node_modules` based on the `package-lock.json` hash to speed up dependency installation.
4. **Install Packages**: Installs the necessary dependencies.
5. **Run Static Checks**: Executes each specified command in the matrix (`lint`, `depcheck`, `check` or others as configured) as defined in the NPM scripts.

This flexible workflow enables dynamic static analysis checks to maintain code quality, making it adaptable to different project requirements.

### [NPM E2E Tests](.github/workflows/e2e-tests-npm.yml)

Executes end-to-end (E2E) tests for an NPM project within a Dockerized environment, supporting optional configurations for TSOA build, database migration, and custom test commands.

#### Inputs

| Input              | Type    | Description                                                                                                             | Default                                                                                           | Required |
|--------------------|---------|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|----------|
| env_vars           | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                                                                                              | false    |
| build_tsoa         | boolean | Specifies if the TSOA build step should be included before running tests                                                | `false`                                                                                           | false    |
| db_migrate         | boolean | Indicates if database migrations should be executed before tests                                                        | `false`                                                                                           | false    |
| docker_compose_file| string  | The Docker Compose file used for building the testing environment                                                       | `docker-compose.yml`                                                                              | false    |
| test_command       | string  | Command used to run E2E tests, which can be customized as needed                                                        | `docker compose -f docker-compose.e2e.yml up --exit-code-from e2e-tests --abort-on-container-exit --quiet-pull` | false    |

#### Workflow Description

This GitHub Actions workflow runs end-to-end tests in a Dockerized environment. It includes custom configurations for TSOA build, database migrations, and specific handling for the `veritable-ui` repository with additional environment variables.

1. **Set Environment Variables**: Parses and applies environment variables from a JSON string.
2. **Docker Buildx Setup**: Configures Docker Buildx to enable advanced Docker build features.
3. **Build E2E Containers**: Builds the necessary containers using Docker Bake, based on the provided Docker Compose file.
4. **Run E2E Tests**:
   - For general repositories, runs the specified E2E `test_command`.
   - For the `veritable-ui` repository, runs the E2E `test_command` with additional environment variables such as `VERITABLE_COMPANY_PROFILE_API_KEY` and `VERITABLE_E2E_OUT_DIR`.
5. **Upload Playwright Report**: Uploads the Playwright report as an artifact, available for 90 days.
6. **Publish CTRF Test Summary**: Publishes a summary of the test results in CTRF format for tracking.

This workflow is highly configurable, allowing users to adapt the E2E test setup for different project requirements while maintaining an efficient and streamlined testing environment.

### [NPM Tests](.github/workflows/tests-npm.yml)

Executes specified NPM test commands (e.g., unit and integration tests) with optional pre-test commands, build steps, and Docker-based dependency setup.

#### Inputs

| Input              | Type    | Description                                                                                                             | Default                          | Required |
|--------------------|---------|-------------------------------------------------------------------------------------------------------------------------|----------------------------------|----------|
| env_vars           | string  | JSON string of environment variables in `key:value` format, parsed and added to `$GITHUB_ENV` at the start of the run | `{}`                             | false    |
| npm_build_command  | string  | Optional NPM build command to execute before running tests                                                              | `''`                             | false    |
| pre_test_command   | string  | Optional command to run before the main test commands                                                                  | `''`                             | false    |
| docker_compose_file| string  | The Docker Compose file used to set up dependencies                                                                     | `docker-compose.yml`             | false    |
| tests              | string  | JSON array of test commands defined in NPM scripts (e.g., `['test:unit', 'test:integration']`)                        | `['test:unit', 'test:integration']` | false    |

#### Workflow Description

This GitHub Actions workflow runs a series of NPM test commands as defined in the `tests` input. The commands are executed in parallel using a matrix strategy, allowing concurrent testing.

1. **Set Environment Variables**: Parses environment variables from a JSON string and sets them in the workflow environment.
2. **Node Setup**: Installs the specified Node.js version.
3. **Node Modules Caching**: Caches `node_modules` to speed up dependency installation, using `package-lock.json` as the cache key.
4. **Install Packages**: Installs project dependencies with `npm ci`.
5. **Build Step (Optional)**: Runs the specified `npm_build_command` if provided.
6. **Environment File Creation**: Creates a `.env` file for environment configuration.
7. **Setup Docker Dependencies**: Brings up services defined in the Docker Compose file to support test dependencies.
8. **Wait for Initialization**: Adds a delay to ensure Docker services are fully started.
9. **Pre-Test Command (Optional)**: Executes the `pre_test_command` if specified.
10. **Run Tests**: Executes each command in the `tests` input matrix (e.g., `test:unit`, `test:integration`) as defined in the NPM scripts.

This workflow provides a flexible setup for running NPM tests, allowing custom commands and Docker-based dependencies to adapt to various testing requirements.

