# Azure DevOps Concepts

* [Azure Pipelines Agents](#azure-pipelines-agents)
  * [Microsoft-hosted Agents](#microsoft-hosted-agents)
  * [Self-hosted Agents](#self-hosted-agents)
  * [Capabilities](#capabilities)
  * [Communication](#communication)
* [Agent Pools](#agent-pools)
* [Self-hosted Windows Agents](#self-hosted-windows-agents)
  * [Download and Configure the Agent](#download-and-configure-the-agent)
  * [Replace an Agent](#replace-an-agent)
  * [Remove and Reconfigure an Agent](#remove-and-reconfigure-an-agent)
  * [Unattended Config](#unattended-config)
* [Artifacts](#artifacts)
  * [Supported Artifact Types](#supported-artifact-types)
* [Build Artifacts](#build-artifacts)
  * [How to Consume Artifacts](#how-to-consume-artifacts)
  * [Build Artifact Tips](#build-artifact-tips)
  * [Related Tasks for Publishing Artifacts](#related-tasks-for-publishing-artifacts)


## Azure Pipelines Agents
[Back to Top](#azure-devops-concepts)  

To build your code or deploy your software using Azure Pipelines, you need at least one agent. As you add more code and people, you'll eventually need more.

When your pipeline runs, the system begins one or more jobs. An agent is installable software that runs one job at a time.

Jobs can be run directly on the host machine of the agent or in a container.

### Microsoft-hosted Agents
[Back to Top](#azure-devops-concepts)  

If your pipelines are in Azure Pipelines, then you've got a convenient option to run your jobs using a **Microsoft-hosted agent**. With microsoft-hosted agents, maintenance and upgrades are taken care of for you. Each time you run a pipeline, you get a fresh virtual machine. The virtual machine is discarded after one use. Microsoft-hosted agents can run jobs directly on the VM or in a container.

> Because we operate on a disconnected environment on a stand-alone instance of Azure DevOps Server, we cannot use Microsoft-hosted agents.

### Self-hosted Agents
[Back to Top](#azure-devops-concepts)  

An agent that you set up and manage on your own to run jobs is a **self-hosted agent**. You can use self-hosted agents in Azure Pipelines or Team Foundation Server (TFS). Self-hosted agents give you more control to install dependent software needed for your builds and deployments. Also, machine-level caches and configuration persist from run to run, which can boost spped.

You can install the agent on Linux, macOS, or Windows machines. You can also install an agent on a Docker container.

### Parallel Jobs
[Back to Top](#azure-devops-concepts)  

YOu can use a parallel job in Azure Pipelines to run a single job at a time in your organization. In Azure Pipelines, you can run parallel jobs on Microsoft-hosted infrastructure or on your own (self-hosted) infrastructure.

> For more information on parallel jobs, see [Parallel jobs in Team Foundation Server](https://docs.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-pipelines-tfs?view=azure-devops-2019&viewFallbackFrom=azure-devops)

### Capabilities
[Back to Top](#azure-devops-concepts)  

Every self-hosted agent has a set of capabilities that indicate what it can do. Capabilities are given name-value pairs that are either automatically discovered by the agent software, in which case they are called **system capabilities**, or those that you define, in which case they are called **user capabilities**.

The agent software automatically determines various system capabilities such as the name of the machine, type of operating system, and versions of certain software installed on the machine. Also, environment variables defined in the machine automatically appear in the list of system capabilities.

When you author a pipeline you specify certain **demands** of the agent. The system sends the job only to agents that have capabilities matching the demands specified in the pipeline. As a result, agent capabilities allow you to direct jobs to specific agents.

> Demands and capabilities apply only to self-hosted agents.

## Agent Pools
[Back to Top](#azure-devops-concepts)  

Instead of managing each agent individually, you organize agents into **agent pools**. In Azure Pipelines, pools are scoped to the entire organization; so you can share the agent machines across projects. In Azure DevOps Server, agent pools are scoped to the entire server; so you can share the agent machines across projects and collections.

## Self-hosted Windows Agents
[Back to Top](#azure-devops-concepts)  

Make sure your machine has these prerequisites:
* Windows 7, 8.1, or 10 (if using a client OS)
* Windows 2008 R2 SP1 or higher (if using a server OS)
* PowerShell 3.0 or higher

Recommended:
* [Visual Studio build tools](https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16) (2015 or higher)
* .NET Framework 4.5 or higher

### Download and Configure the Agent
[Back to Top](#azure-devops-concepts)  

1. The latest agent package can be downloaded from the [Azure Pipelines Agent](https://github.com/microsoft/azure-pipelines-agent/releases) repository's releases section.
2. Unzip the agent package at a location where you want the agent to be managed.
    * We are unzipping it into a directory at `C:\DevOpsAgent`
3. From an elevated PowerShell terminal, run `.\config.cmd`. This will ask you a series of questions for configuring the agent.
    * You will need to know the URL of the DevOps server. For instance: http://azuredevops.domain.com
    * You will use **Integrated** authentication.
    * You will also need to have a user account and password for a service account if you are configuring the agent as a service.

If you configured the agent to run as a service, it starts automatically. You can view and control the agent running status from the services snap-in. Run `services.msc` and look for one of:

* "Azure Pipelines Agent (*name of your agent*)"
* "VSTS Agent (*name of your agent*)"
* "vstsagent.(*organization name*).(*name of your agent*)"

### Replace an Agent
[Back to Top](#azure-devops-concepts)  

To replace an agent, follow the *Download and configure the agent* steps again.

When you configure an agent using the same name as an agent that already exists, you're asked if you want to replace the existing agent. If you answer `Y`, then make sure you remove the agent (see below) that you're replacing. Otherwise, after a few minutes of conflicts, one of the agents will shut down.

### Remove and Reconfigure an Agent
[Back to Top](#azure-devops-concepts)  

To remove the agent, execute `.\config.cmd remove` from an elevated PowerShell terminal where you extracted the agent package.

### Unattended Config
[Back to Top](#azure-devops-concepts)  

The agent can be set up from a script with no human intervention. You must pass `--unattended` and the answers to all questions.

To configure an agent, it must know the URL to your organization or collection and credentials of someone authorized to set up agents. All other responses are optional. Any command-line parameter cna be specified using an environment variable instead: put its name in upper case and prepend `VSTS_AGENT_INPUT_`. For example, `VSTS_AGENT_INPUT_PASSWORD` instead of specifying `--password`.

#### Required Options

* `--unattended` - agent setup will not prompt for information, and all settings must be provided on the command line
* `--url <url>` - URL of the server. For example: https://dev.azure.com/myorganization or http://my-devops-server.domain.net
* `--auth <type>` - authentication type. Valid valuse are:
  * `pat` (Personal access token)
  * `negotiate` (Kerberos or NTLM)
  * `alt` (Basic authentication)
  * `integrated` (Windows default credentials)

#### Authentication Options

* If you chose `--auth pat`:
  * `--token <token>` - specifies your personal access token
* If you chose `--auth negotiate` or `--auth alt`:
  * `--userName <userName>` - specifies a Windows username in the format `domain\userName` or `userName@domain.com`
  * `--password <password>` - specifies a password

#### Pool and agent names

* `--pool <pool>` - pool name for the agent to join
* `--agent <agent>` - agent name
* `--replace` - replace the agent in a pool. If another agent is listening by the same name, it will start failing with a conflict.

#### Agent setup

* `--work <workDirectory>` - work directory where job data is stored. Defaults to `_work` under the root of the agent directory. The work directory is owned by a given agent and should not share between multiple agents.
* `--acceptTeeEula` - accept the Team Explorer Everywhere End User License Agreement (macOS and Linux only)
* `--once` - accept only one job and then spin down gracefully (useful for running a service like Azure Container Instances)

#### Windows-only startup

* `--runAsService` - configure the agent to run as a Windows service (requires administrator permission)
* `--runAsAutoLogon` - configure auto-logon and run the agent on startup (requires administrator permission)
* `--windowsLogonAccount <account>` - used with `--runAsService` or `--runAsAutoLogon` to specify the Windows user name in the format `domain\userName` or `userName@domain.com`
* `--windowsLogonPassword <password>` - used with `--runAsService` or `--runAsAutoLogon` to specify Windows logon password
* `--overwriteAutoLogon` - used with `--runAsAutoLogon` to overwrite the existing auto logon on the machine
* `--noRestart` - used wtih `--runAsAutoLogon` to stop the host from restarting after agent configuration completes

#### Deployment group only

* `--deploymentGroup` - configure the agent as a deployment group agent
* `--deploymentGroupName <name>` - used with `--deploymentGroup` to specify the deployment group for the agent to join
* `--projectName <name>` - used with `--deploymentGroup` to set the project name
* `--addDeploymentGroupTags` - used with `--deploymentGroup` to indicate that deployment group tags should be added
* `--deploymentGroupTags <tags>` - used with `--addDeploymentGroupTags` to specify the comma-separated list of tags for the deployment group agent - for example `web, db`

`.\config --help` always lists the latest required and optional responses.

## Artifacts
[Back to Top](#azure-devops-concepts)  

You can publish and consume many types of packages and artifacts with Azure Pipelines. Your continuous integration / continuous deployment (CI / CD) pipeline can publish specific package types to their respective package repositories (NuGet, npm, Python, and so on). Or you can use build artifacts and pipeline artifacts to help store build outputs and intermediate files between build steps. YOu can then add onto, build, test, or even deploy those artifacts.

### Supported Artifact Types
[Back to Top](#azure-devops-concepts)  

Artifact | Description
---------|------------
**Build Artifacts** | Build artifacts are the files that you want your build to produce. Build artifacts can be nearly anything that your team needs to test or deploy your app. For example, you've got `.dll` and `.exe` files and a `.PDB` symbols file of a C# or C++ .NET Windows app.
**Pipeline Artifacts** | You can use pipeline artifacts to help store build outputs and move intermediate files between jobs in your pipeline. Pipeline artifacts are tied to the pipeline that they're created in. You can use them within the pipeline and download them from the build, as long as the build is retained. Pipeline artifacts are the new generation of build artifats. They take advantage of existing services to dramatically reduce the time it takes to store outputs from your pipelines. **Only available in Azure DevOps Services (no on-prem)**.
**Maven** | You can publish Maven artifacts to Azure Artifacts feeds or Maven repositories.
**npm** | You can publish npm packages to Azure Artifacts or npm registries.
**NuGet** | You can publish NuGet packages to Azure Artifacts, other NuGet services (like NuGet.org), or internal NuGet repositories.
**PyPl** | YOu can publish Python packages to Azure Artifacts or PyPl repositories.
**Symbols** | Symbol files contain debugging information for compiled executables. You can publish symbols to symbol servers. Symbol servers enable debuggers to automatically retrieve the correct symbol files without knowing specific product, package, or build information.
**Universal** | Universal pakcages store one ore more files together in a single unit that has a name and version. Unlike pipeline artifacts that reside in the pipeline, Universal Packages residew thin a feed in Azure Artifacts.

> Build and Release artifacts will be available as long as that Build or Release run is retained, unless you specify how long to retain the artifacts.

## Build Artifacts
[Back to Top](#azure-devops-concepts)  

Artifacts are the files that you want your build to produce. Artifacts can be anything that your team needs to test or deploy your app.

Articats can be published at any stage of pipeline. You can use two methods for configuring what to publish as an artifact and when to publish it: alongside your code with **YAML**, or in the Azure Pipelines UI with the **classic editor**.

**Example: Publish a text file as an artifact**  

```yml
- powershell: gci env:* | sort-object name | Format-Table -AutoSize | Out-File $env:BUILD_ARTIFACTSTAGINGDIRECTORY/environment-variables.txt

- task: PublishBuildArtifacts@1
  inputs:
    pathtoPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: drop
```

**Example: Publish two sets of artifacts**  

```yml
- powershell: gci env:* | sort-object name | Format-Table -AutoSize | Out-File $env:BUILD_ARTIFACTSTAGINGDIRECTORY/environment-variables.txt

- task: PublishBuildArtifacts@1
  inputs:
    pathtoPublish: '$(BUild.ArtifactStagingDirectory)'
    artifactName: drop1
- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: drop2
```

**Example: Assemble C++ artifacts into one location and publish as an artifact**

```yml
- powershell: gci env:* | sort-object name | Format-Table -AutoSize | Out-File $env:BUILD_ARTIFACTSTAGINGDIRECTORY/environment-variables.txt

- task: CopyFiles@2
  inputs: 
    sourceFolder: '$(Build.SourcesDirectory)'
    contents: '**/$(BuildConfiguration)/**/?(*.exe|*.dll|*.pdb)'
    TargetFolder: '$(Build.ArtifactStagingDirectory)'
- task: PublishBuildArtifacts@1
  inputs:
    pathtoPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: drop
```

### How to Consume Artifacts
[Back to Top](#azure-devops-concepts)  

**Consume artifacts in release pipelines**  

You can download artifacts produced by either a build pipeline (created in a classic editor) or a YAML pipeline (created through a YAML file) in a release pipeline and deploy them to the target of your choice. At present, you cannot download an artifact produced by a YAML pipeline in another YAML pipeline.

**Consume an artifact in the next job of your pipeline**  

You can consume an artifact produced by one job in a subsequent job of the pipeline, even when that job is in a different stage (YAML pipelines). THis can be useful to test your artifact.

**Download to debug**  

You can download an artifact directly from a pipeline for use in debugging.

```yml
- powershell: gci env:* | sort-object name | Format-Table -AutoSize | Out-File $env:BUILD_ARTIFACTSTAGINGDIRECTORY/environment-variables.txt

- task: DownloadBuildArtifacts@0
  inputs:
    buildType: 'current'
    downloadType: 'single'
    artifactName: 'drop'
    downloadPath: '$(System.ArtifactsDirectory)'
```

### Build Artifact Tips
[Back to Top](#azure-devops-concepts)  

* **Artifact publish location** argument: **Azure Pipelines/TFS (TFS 2018 RTM and older**: Artifact type: Server) is the best and simplest choice in most cases. This choice causes the artifacts to be stored in Azure Pipelines or TFS. But if you're using a private Windows agent, you've got the option to drop to a UNC file share.
* **Artifact name** argument: Just enter a name that's meaningful to you.
* Use forward slashes in file path arguments so that they work for all agents. Backslashes don't work for macOS and Linux agents.
* On Azure Pipelines and some versions of TFS, two different [variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml) point to the staging directory: `Build.ArtifactStagingDirectory` and `Build.StagingDirectory`. These are interchangeable.
* The directory referenced by `Build.ArtifactStagingDirectory` is cleaned up after each build.
* You can [get build artifacts from the REST API](https://docs.microsoft.com/en-us/rest/api/azure/devops/build/Artifacts?view=azure-devops-rest-5.0).

### Related Tasks for Publishing Artifacts
[Back to Top](#azure-devops-concepts)  

Use these tasks to publish artifacts:
* [Utility: Copy Files](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/copy-files?view=azure-devops) By copying files to `$(Build.ArtifactStagingDirectory)`, you can publish multiple files of different types from different places specified by your [matching patterns](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/file-matching-patterns?view=azure-devops).
* [Utility: Delete Files](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/delete-files?view=azure-devops) You can prune unnecessary files that you copied to the staging directory.
* [Utility: Publish Build Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-build-artifacts?view=azure-devops)

When a build is done, if you watched it run, select the **Summary** tab and see your artifact in the **Build artifacts published** section.