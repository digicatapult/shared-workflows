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

Builds a Docker container and optionally pushes it to GitHub Container Registry (GHCR), DockerHub, or both, based on specified input parameters.

#### Inputs

| Input            | Type    | Description                                                                                                             | Default                     | Required |
|------------------|---------|-------------------------------------------------------------------------------------------------------------------------|-----------------------------|----------|
| env_vars         | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                        | false    |
| push_dockerhub   | boolean | Specifies whether to push the built image to DockerHub                                                                  | `false`                     | false    |
| push_ghcr        | boolean | Specifies whether to push the built image to GHCR                                                                       | `false`                     | false    |
| docker_platforms | string  | Specifies the architectures to build containers for                                                                     | `"linux/amd64,linux/arm64"` | false    |
| docker_file      | string  | Specifies the Dockerfile to be used for building the image                                                              | `Dockerfile`                | false    |

#### Secrets

| Secret              | Description                                                                                           |
|---------------------|-------------------------------------------------------------------------------------------------------|
| GITHUB_TOKEN        | Token used for authenticating with GitHub to access and push images to GHCR                           |
| DOCKERHUB_USERNAME  | DockerHub username used for authentication when pushing to DockerHub                                  |
| DOCKERHUB_TOKEN     | DockerHub token (or password) used for authentication when pushing to DockerHub                       |

#### Workflow Description

This GitHub Actions workflow builds a Docker container based on the specified Dockerfile and configuration. It can push the built image to either GitHub Container Registry (GHCR) or DockerHub (or both) if the respective options (`push_dockerhub` and `push_ghcr`) are enabled. The workflow supports multi-platform builds and version tagging using the `digicatapult/check-version` action to manage version information. It also includes four main build/push cases:

1. **GHCR Only**: Builds and pushes the Docker image only to GHCR.
2. **DockerHub Only**: Builds and pushes the Docker image only to DockerHub.
3. **Both GHCR and DockerHub**: Builds and pushes the Docker image to both registries.
4. **Build Only**: Builds the Docker image without pushing it to any registry.

Additionally, the workflow applies Open Container Initiative (OCI) labels to the images for enhanced metadata tracking.

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
| AUTH_TOKEN   | Authentication token required to publish the package to the NPM registry      |

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
| matrix_commands | string  | A JSON array of commands to run in the static checks matrix, each representing an NPM script defined in the package     | `[lint,depcheck,xss-scan,check]` | false    |

#### Workflow Description

This GitHub Actions workflow runs a series of static checks on an NPM project based on a configurable list of commands. The commands run in parallel through a matrix strategy, allowing selective failure without interrupting the entire workflow.

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Node Setup**: Configures the Node.js environment.
3. **Node Modules Caching**: Caches `node_modules` based on the `package-lock.json` hash to speed up dependency installation.
4. **Install Packages**: Installs the necessary dependencies.
5. **Run Static Checks**: Executes each specified command in the matrix (`lint`, `depcheck`, `xss-scan`, `check` or others as configured) as defined in the NPM scripts.

This flexible workflow enables dynamic static analysis checks to maintain code quality, making it adaptable to different project requirements.

### [NPM E2E Tests](.github/workflows/tests-e2e-npm.yml)

Runs end-to-end (e2e) tests for an NPM project using Docker Compose, with optional setup steps for TSOA build, database migration, and project-specific configurations.

#### Inputs

| Input              | Type    | Description                                                                                                             | Default               | Required |
|--------------------|---------|-------------------------------------------------------------------------------------------------------------------------|-----------------------|----------|
| env_vars           | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                  | false    |
| build_tsoa         | boolean | Specifies if the TSOA build step should be included before running tests                                                | `false`               | false    |
| db_migrate         | boolean | Indicates if database migrations should be executed before tests                                                        | `false`               | false    |
| docker_compose_file| string  | The Docker Compose file to use for setting up the testing environment                                                   | `docker-compose.yml`  | false    |

#### Workflow Description

This GitHub Actions workflow runs end-to-end tests within a Dockerized environment, leveraging Docker Compose for test setup. The workflow is adaptable, providing tailored handling for the `veritable-ui` repository with additional environment variables.

1. **Setting Environment Variables**: Parses and sets environment variables from a JSON string.
2. **Docker Buildx Setup**: Configures Docker Buildx to support advanced Docker build functionality.
3. **Build E2E Containers**: Builds the necessary Docker containers using Docker Bake based on the specified Docker Compose file.
4. **Run E2E Tests**:
   - For general repositories, runs e2e tests with the specified Docker Compose configuration.
   - For the `veritable-ui` repository, sets additional environment variables specific to its requirements, such as `VERITABLE_COMPANY_PROFILE_API_KEY` and `VERITABLE_E2E_OUT_DIR`.
5. **Upload Playwright Report**: Uploads the Playwright report as an artifact, available for 90 days, for test reporting and tracking.
6. **Publish CTRF Test Summary**: Publishes a summary of the test results in CTRF format for easy tracking and visibility.

This workflow provides a robust, customizable e2e testing setup, supporting different configurations and dependency management to suit various project needs.

### [NPM Tests](.github/workflows/tests-npm.yml)

Executes specified NPM test commands (e.g., unit and integration tests) with optional steps for TSOA build, database migrations, and Docker-based dependency setup.

#### Inputs

| Input              | Type    | Description                                                                                                             | Default                          | Required |
|--------------------|---------|-------------------------------------------------------------------------------------------------------------------------|----------------------------------|----------|
| env_vars           | string  | A JSON string representing environment variables in the format `key:value`; parsed and added to `$GITHUB_ENV` at the beginning of the run | `{}`                             | false    |
| build_tsoa         | boolean | Specifies if the TSOA build step should be run before tests                                                             | `false`                          | false    |
| db_migrate         | boolean | Indicates if database migrations should be run before tests                                                             | `false`                          | false    |
| docker_compose_file| string  | The Docker Compose file to use for setting up dependencies                                                              | `docker-compose.yml`             | false    |
| tests              | string  | A JSON array of test commands to run as defined in the NPM scripts (e.g., `['test:unit', 'test:integration']`)         | `['test:unit', 'test:integration']` | false    |

#### Workflow Description

This GitHub Actions workflow runs a configurable set of NPM test commands specified in the `tests` input. The commands execute in parallel through a matrix strategy, allowing multiple tests to run concurrently.

1. **Setting Environment Variables**: Parses and applies environment variables from a JSON string.
2. **Node Setup**: Configures the Node.js environment.
3. **Node Modules Caching**: Caches `node_modules` based on the `package-lock.json` hash to speed up dependency installation.
4. **Install Packages**: Installs project dependencies with `npm ci`.
5. **TSOA Build (Optional)**: Executes `tsoa:build` if `build_tsoa` is set to true.
6. **Environment File Creation**: Creates a `.env` file if needed for configuration.
7. **Setup Dependencies with Docker**: Brings up services defined in the specified Docker Compose file to support test dependencies.
8. **Wait for Dependency Initialization**: Introduces a delay to ensure Docker services are fully initialized.
9. **Database Migrations (Optional)**: Runs `db:migrate` if `db_migrate` is set to true.
10. **Run Tests**: Executes each test command specified in the `tests` input matrix (e.g., `test:unit`, `test:integration`).

This workflow enables flexible and efficient testing setups, supporting custom configurations and dependencies to streamline project testing requirements.
