# Build and Deploy

## Overview

This workflow builds a .NET application, runs tests, packages it as a Docker image, and deploys it to Azure Container Apps in staging and production environments. It runs automatically on every push or pull request targeting the `main` branch, and can also be triggered manually.

## Triggers

- **Push to `main`**: Runs the full workflow whenever code is pushed to the `main` branch.
- **Pull request to `main`**: Runs the Build job only (deployment steps are skipped) when a pull request targets the `main` branch.
- **Manual dispatch (`workflow_dispatch`)**: Can be triggered manually from the GitHub Actions UI.

## Environment Variables

| Variable | Value | Description |
|---|---|---|
| `DOTNET_VERSION` | `10.0.x` | The .NET SDK version used to build the project. |
| `PROJECT_PATH` | `MinimalApi/MinimalApi.csproj` | Path to the main .NET project file. |
| `TEST_PROJECT_PATH` | `MinimalApi/MinimalApi.csproj` | Path to the test project (currently the same as the main project; update when a dedicated test project is added). |
| `DOCKER_IMAGE_NAME` | `minimalapi` | Name used for the Docker image. |
| `AZURE_CONTAINER_REGISTRY` | (from secret) | The Azure Container Registry hostname, read from the `AZURE_CONTAINER_REGISTRY` repository secret. |

## Jobs

### Build

**Runs on**: `ubuntu-latest`

**Needs**: None

**Condition**: Always runs

**Permissions**: `contents: read`

#### Steps

1. **Checkout code**: Checks out the repository source code into the runner so subsequent steps can access it. Uses `actions/checkout@v4`.

2. **Setup .NET**: Installs the .NET SDK version specified by `DOTNET_VERSION` (`10.0.x`). Uses `actions/setup-dotnet@v4`.

3. **Restore dependencies**: Downloads all NuGet packages required by the project. Runs `dotnet restore` on the main project file.

4. **Build**: Compiles the application in `Release` configuration without re-restoring dependencies (since they were restored in the previous step). Runs `dotnet build --no-restore --configuration Release`.

5. **Test**: Runs the test suite in `Release` configuration. Runs `dotnet test --no-build --configuration Release --verbosity normal`.

6. **Build Docker image**: Builds a Docker image from the `MinimalApi/` directory, tagging it with the current commit SHA (`github.sha`). Runs `docker build`.

7. **Log in to Azure Container Registry**: Authenticates to the Azure Container Registry using credentials stored in repository secrets (`AZURE_REGISTRY_USERNAME` and `AZURE_REGISTRY_PASSWORD`). Uses `docker/login-action@v3`. **Only runs on non-pull-request events** (skipped for pull requests).

8. **Push Docker image to Azure Container Registry**: Tags the Docker image with both the commit SHA and `latest`, then pushes both tags to the Azure Container Registry. **Only runs on non-pull-request events** (skipped for pull requests).

---

### Staging

**Runs on**: `ubuntu-latest`

**Needs**: `build` (runs after the Build job succeeds)

**Condition**: Only runs on **non-pull-request** events (skipped for pull requests)

**Permissions**: None (uses `{}`)

**Environment**: `staging` — uses GitHub environment protection rules; the deployment URL is available as an output.

#### Steps

1. **Log in to Azure**: Authenticates to Microsoft Azure using the `AZURE_CREDENTIALS` repository secret. Uses `azure/login@v2`.

2. **Deploy to Azure Container Apps (Staging)**: Deploys the Docker image (tagged with the current commit SHA) to the Azure Container App configured for staging. Uses `azure/container-apps-deploy-action@v2` with the staging container app name and resource group from repository secrets.

---

### Production

**Runs on**: `ubuntu-latest`

**Needs**: `staging` (runs after the Staging job succeeds)

**Condition**: Always runs (after staging succeeds and is not a pull request)

**Permissions**: None (uses `{}`)

**Environment**: `production` — uses GitHub environment protection rules; the deployment URL is available as an output.

#### Steps

1. **Log in to Azure**: Authenticates to Microsoft Azure using the `AZURE_CREDENTIALS` repository secret. Uses `azure/login@v2`.

2. **Deploy to Azure Container Apps (Production)**: Deploys the Docker image (tagged with the current commit SHA) to the Azure Container App configured for production. Uses `azure/container-apps-deploy-action@v2` with the production container app name and resource group from repository secrets.
