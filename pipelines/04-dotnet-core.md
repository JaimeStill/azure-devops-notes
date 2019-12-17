# .NET Core Pipeline Ecosystem  

* [Build Environment](#build-environment)
* [Restore Dependencies](#restore-dependencies)
* [Build Your Project](#build-your-project)
* [Run Your Tests](#run-your-tests)
* [Collect Code Coverage](#collect-code-coverage)
* [Package and Deliver Your Code](#package-and-deliver-your-code)
* [Troubleshooting](#troubleshooting)

Use a pipeline to automatically build and test your .NET Core projects. After those steps are done, you can then deploy or publish your project.

## Build Environment
[Back to Top](#net-core-pipeline-ecosystem)  

> Due to the fact that we're in an offline environment, [Microsoft-hosted agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops) are not available and the steps in this section are irrelevant. It is included here for completeness. See the note following this section regarding self-hosted agents.

You can use Azure Pipelines to build your .NET Core projects on Windows, Linux, or macOS without needing to set up any infrastructure of your own. The [Microsoft-hosted agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops) in Azure Pipelines have several released versions of the .NET Core SDKs preinstalled.

```yml
pool:
  vmImage: 'ubuntu-16.04' # examples of other options: 'macOS-10.13', 'vs2017-win2016'
```

The Microsoft-hosted agents don't include some of the older versions of the .NET Core SDK. They also don't typically include prerelease versions. If you need these kinds of SDKs on Microsoft-hosted agents, add the **.NET Core Tool Installer** task to the beginning of the process.

```yml
steps:
  - task: UseDotNet@2
    inputs:
      version: '3.0.x'

  - task: UseDotNet@2
    inputs:
      version: '2.2.x'
      packageType: runtime
```

If you are installing on a Windows agent, it will already have a .NET Core runtime on it. TO install a newer SDK, set `performMultiLevelLookup` to `true`:

```yml
steps:
  - task: UseDotNet@2
    displayName: 'Install .NET Core SDK'
    inputs:
      version: '3.0.x'
      performMultiLevelLookup: true
```

> As an alternative, you can set up a [self-hosted agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops#install) and save the cost of running the tool installer. You can use self-hosted agents to save additional time if you have a large repository or you run incremental builds. A self-hosted agent can also help you in using the preview or private SDKs that are not officially supported by Azure DevOps or you have available on your corporate or on-premises environments only.

## Restore Dependencies
[Back to Top](#net-core-pipeline-ecosystem)  

NuGet is a popular way to depend on code that you don't build. You can download NuGet packages by runing the `dotnet restore` command either through the [.NET Core](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops) task or direcly in a script in your pipeline.

If your builds occasionally fail when restoring packages from NuGet.org due to connection issues, you can use Azure Artifacts in conjunction with [upstream sources](https://docs.microsoft.com/en-us/azure/devops/artifacts/concepts/upstream-sources?view=azure-devops) and cache the packages. The credentials of the pipeline are automatically used when connecting to Azure Artifacts. These credentials are typically derived from the **Project Collection Build Service** account.

If you want to specify a NuGet repository, put the URLs in a `NuGet.config` file in your repository. If your feed is authenticated, manage its credentials by creating a NuGet service connection in the **Services** tab under **Project Settings**.

To restore packages from a custom feed:

```yml
# do this before your build tasks
steps:
  - task: DotNetCoreCLI@2
    inputs:
      command: restore
      projects: '**/*.csproj'
      feedsToUse: config
      nugetConfigPath: NuGet.config   # Relative to root of the repository
      externalFeedCredentials: <Name of the NuGet service connection>
```

> For more information about NuGet service connections, see [publish to NuGet feeds](https://docs.microsoft.com/en-us/azure/devops/pipelines/artifacts/nuget?view=azure-devops).

## Build Your Project
[Back to Top](#net-core-pipeline-ecosystem)  

You build your .NET Core project either by running the `dotnet build` command in your pipeline or by using the .NET Core task.

To build your project using the .NET Core task:

```yml
steps:
  - task: DotNetCoreCLI@2
    displayName: Build
    inputs:
      command: build
      projects: '**/*.csproj'
      arguments: '--configuration Release'    # Update this to match your need
```

You can run any custom dotnet command in your pipeline:

```yml
steps:
  - task: DotNetCoreCLI@2
  displayName: 'Install dotnetsay'
  inputs:
    command: custom
    custom: tool
    arguments: 'install -g dotnetsay'
```

## Run Your Tests
[Back to Top](#net-core-pipeline-ecosystem)  

If you have test projects in your repository, then use the **.NET Core** task to run unit tests by using testing frameworks like MSTest, xUnit, and NUnit. For this functionality, the test project must reference [Microsoft.NET.Test.SDK](https://www.nuget.org/packages/Microsoft.NET.Test.SDK) version 15.8.0 or higher. Test results are automatically published to the service. These results are then made available to you in the build summary and can be used for troubleshooting failed tests and test-timing analysis.

```yml
steps:
# ...
# do this after other tasks such as building
  - task: DotNetCoreCLI@2
    inputs:
      command: test
      projects: '**/*Tests/*.csproj'
      arguments: '--configuration $(buildConfiguration)'
```

An alternative is to run the `dotnet test` command with a specific logger and then use the **Publish Test Results** task:

```yml
steps:
# ...
# do this after your tests have run
  - script: dotnet test <test-project> --logger trx
  - task: PublishTestResults@2
    condition: succeededOrFailed()
    inputs:
      testRunner: VSTest
      testResultsFiles: '**/*.trx'
```

## Collect Code Coverage
[Back to Top](#net-core-pipeline-ecosystem)  

If you're building on the Windows platform, code coverage metrics can be collected by using the built-in coverage data collector. For this functionality, the test project must reference [MIcrosoft.NET.Test.SDK](https://www.nuget.org/packages/Microsoft.NET.Test.SDK) version 15.8.0 or higher. If you use the **.NET Core** task to run tests, coverage data is automatically published to the server. The **.coverage** file can be downloaded from the build summary for viewing in Visual Studio.

```yml
steps:
# ...
# do this after other tasks such as building
- task: DotNetCoreCLI@2
  inputs:
    command: test
    projects: '**/*Tests/*.csproj'
    arguments: '--configuration $(buildConfiguration) --collect "Code coverage"'
```

If you choose to run the `dotnet test` command, specify the test results logger and coverage options. Then use the [Publish Test Results](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/test/publish-test-results?view=azure-devops) task:

```yml
steps:
# ...
# do this after your tests have run
- script: dotnet test <test-project> --logger trx --collect "Code coverage"
- task: PublishTestResults@2
  inputs:
    testRunner: VSTest
    testResultsFiles: '**/*.trx'
```

## Package and Deliver Your Code
[Back to Top](#net-core-pipeline-ecosystem)  

After you've built and tested your app, you can upload the build output to Azure Pipelines or TFS, create and publish a NuGet package, or package the build intoa  .zip to be deployed to a web application.

### Publish artifacts to Azure Pipelines  

To publish the output of your .NET **build**,

* Run `dotnet publish --output $(Build.ArtifactStagingDirectory)` on CLI or add the **DotNetCoreCLI@2** task with publish command.
* Publish the artifact by using the **PublishBuildArtifacts** task.

```yml
steps:
- task: DotNetCoreCLI@2
  inputs:
    command: publish
    publisWebProjects: true
    arguments: '--configuration $(BuildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true

# this code taks all the files in $(Build.ArtifactStagingDirectory) and uploads them as an artifact of your build.
- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: 'myWebsiteName'
```

> To copy additional files to Build directory before publishing, use [Utility: copy files](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/copy-files?view=azure-devops).

### Publish to a NuGet Feed

```yml
steps:
# ...
# do this near the end of your pipeline in most cases
- script: dotnet pack /p:PackageVersion=$(version) # define version variable elsewhere
- task: NuGetAuthenticate@0
  input:
    nuGetServiceConnections: '<Name of the NuGet service connection>'
- task: NuGetCommand@2
  inputs:
    command: push
    nuGetFeedType: external
    publishFeedCredentials: '<Name of the NuGet service connection>'
    versioningScheme: byEnvVar
    versionEnvVar: version
```

### Deploy a Web App

To create a .zip file archive that's ready for publishing to a web app, add the following snippet:

```yml
steps:
# ...
# do this after you've built your app, near the end of your pipeline in most cases
# for example, you do this before you deploy to an Azure web app on Windows
- task: DotNetCoreCLI@2
  inputs:
    command: publish
    publishWebProjects: true
    arguments: '--configuration $(BuildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true
```

## Troubleshooting
[Back to Top](#net-core-pipeline-ecosystem)  

If you're able to build your project on your development machine, but you're having trouble building it on Azure Pipelines or TFS, explore the following potential causes and corrective actions:

* We don't install prerelease versions of .NET Core SDK on Microsoft-hosted agents. After a new version of the .NET Core SDK is released, it can take a few weeks for us to roll it out to all the datacenters that Azure Pipelines runs on. YOu don't have to wait for us to finish this rollout. You can use the **.NET Core Tool Installer**, as explained in the notes above, to install the desired version of the .NET Core SDK on Microsoft-hosted agents.

* Check that the versions of the .NET Core SDK and runtime on your development machine match those on the agent. You can either include a command-line script `dotnet --version` in your pipeline to print the version of the .NET Core SDK. Either use the **.NET Core TOol Installer** to deploy the same version on the agent, or update your projects and development machine to the newer version of the .NET Core SDK.

* You might be using some logic in the Visual Studio IDE that isn't encoded in your pipeline. Azure Pipelines or TFS runs each of the commands you specify in the tasks one after the other in a new process. Look at the logs from the Azure Pipelines or TFS build to see the exact commands that ran as part of the build. Repeat the same commands in the same order on your development machine to locate the problem.

* If you have a mixed solution that includes som .NET Core projects and some .NET Framework projects, you should also use the **NuGet** task to restore packages specified in `packages.config` files. Similarly, you should add **MSBuild** or **Visual Studio Build** tasks to build the .NET Framework projects.

* If your build fails intermittently while restoring packages, either NuGet.org is having issues, or there are networking problems between the Azure datacenter and NuGet.org. These aren't under our control, and you might need to explore whether using Azure Artifacts with NuGet.org as an upstream source improves the reliability of your builds.

* Occasionally, when we roll out an update to the hosted images with a new version of the .NET Core SDK or Visual STudio, something might break your build. This can happen, for example, if a newer version or feature of the NuGet tool is shipped with the SDK. To isolate these problems, use the **.NET Core Tool Installer** task to specify the version of the .NET Core SDK that's used in your build.