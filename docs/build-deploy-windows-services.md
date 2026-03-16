# .Net Full Framework Build and Release to Windows Services

## Overview

This workflow builds a .NET Full Framework solution and deploys it as a Windows Service on a self-hosted runner. It compiles the project using MSBuild, publishes the build artifacts, and then stops the existing Windows Service, replaces its files, and restarts it.

## Triggers

- **Push to `master`**: Runs automatically whenever code is pushed to the `master` branch.
- **Pull request targeting `master`**: Runs automatically when a pull request is opened or updated against the `master` branch.
- **Manual trigger**: Can be triggered manually via the GitHub Actions UI (`workflow_dispatch`).

## Environment Variables

| Variable | Value | Description |
|---|---|---|
| `OUTPUT_DIRECTORY` | `c:\myapp` | Directory where the build output is placed. |
| `SERVICE_NAME` | `SampleWindowsServicesGHA` | The name of the Windows Service to manage. |
| `SERVICE_DIRECTORYNAME` | `C:\Services` | The directory on the server where the service files live. |
| `DOWNLOAD_DIRECTORY` | `C:\ServicesDownload` | Temporary directory used to download build artifacts before deployment. |

## Jobs

### build

**Runs on**: `self-hosted`

**Needs**: None

**Condition**: Always runs

**Permissions**: Default

#### Steps

1. **Checkout**: Checks out the repository source code onto the runner using `actions/checkout@v2`, making the code available for subsequent steps.
2. **Setup NuGet.exe for use with actions**: Installs and configures the NuGet CLI (`NuGet/setup-nuget@v1.0.5`) so packages can be restored in the next step.
3. **Restore Packages**: Runs `nuget restore` to download and restore all NuGet dependencies defined in the solution file (`SampleWindowsServicesGHA.sln`).
4. **setup-msbuild**: Installs and adds MSBuild to the PATH using `microsoft/setup-msbuild@v1.0.2`, making it available for the build step.
5. **Build using MSBuild**: Compiles the solution using MSBuild with the `Release` configuration targeting `Any CPU`, and outputs the build artifacts to `c:\myapp`.
6. **Publish Artifacts**: Uploads the build output from `c:\myapp` as a workflow artifact named `windowsservices` using `actions/upload-artifact@v1.0.0`, making it available for the deploy job.

---

### deploy

**Runs on**: `self-hosted`

**Needs**: `build` (this job only starts after the `build` job completes successfully)

**Condition**: Always runs

**Permissions**: Default

#### Steps

1. **Download a Build Artifact**: Downloads the `windowsservices` artifact produced by the `build` job into `C:\ServicesDownload\SampleWindowsServicesGHA` using `actions/download-artifact@v2.0.7`.
2. **List Windows Services SampleWindowsServicesGHA**: Runs a PowerShell command (`Get-Service`) to verify the Windows Service exists and display its current status.
3. **Stop Windows Services SampleWindowsServicesGHA**: Stops the Windows Service by running `Stop-Service` in an elevated PowerShell process, then waits in a loop until the service status reports `Stopped`.
4. **Copy Windows Services files**: Copies the newly downloaded build files from the download directory into the service's installation directory (`C:\Services\SampleWindowsServicesGHA`), overwriting existing files.
5. **Start Windows Services SampleWindowsServicesGHA**: Starts the Windows Service by running `Start-Service` in an elevated PowerShell process, then waits in a loop until the service status reports `Running`.
