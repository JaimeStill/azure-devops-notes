# Pipeline Confiuration

* [Agent Pool](#agent-pool)
* [Pipelines Agent Script Package](#pipelines-agent-script-package)
* [Agent Configuration](#agent-configuration)
* [Deployment Group Configuration](#deployment-group-configuration)
* [Deployment Agent Configuration](#deployment-agent-configuration)
* [Build Pipeline Setup](#build-pipeline-setup)
* [New Build Pipeline](#new-build-pipeline)
* [Release Pipeline Setup](#release-pipeline-setup)

The intent of this document is to capture the process of setting up CI / CD for a .NET Core / Angular Web App on a closed network with no internet.

> If any of these concepts are unclear, refer to the [Azure DevOps Documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines)

## Agent Pool
[Back to Top](#pipeline-configuration)

The **Default** agent pool is available to each project automatically. This allocates many agents to execute build pipelines.

To view the agent pools, click the **Project Settings** link in the **Project** view, then click the **Agent Pools** tab.

## Pipelines Agent Script Package
[Back to Top](#pipeline-configuration)

The latest agent package can be acquired on the [azure-pipelines-agent repository](https://github.com/microsoft/azure-pipelines-agent/releases).

1. Download and extract the latest agent to a folder named `DevOpsAgent`.
2. Create your own `create-dep-agent` script in the extracted directory:  

Argument | Description
---------|------------
`devops-instance` | The URL your Azure DevOps instance resolves to.
`collection-name` | The collection your project is hosted in.
`project-name` | The project the deployment agent is being built for.  

`create-dep-agent.cmd`  
```pwsh
.\config.cmd --deploymentgroup --deploymentgroupname "IIS" --agent $env:COMPUTERNAME --runasservice --work '_work' --url 'http://{devops-instance}` --collectionname '{collection-name}' --projectname '{project-name}' --auth Integrated
```  

3. Create your own `remove-agent` script in the extracted directory:  

`remove-agent.cmd`  
```pwsh
.\config.cmd remove
```  

4. Place the `DevOpsAgent` folder in the location on your **IIS VM** that you want to act as the agent host.
    * This is the directory where all file interactions will take place, self-contained in the `DevOpsAgent` folder.
    * Our convention is `C:\DevOpsAgent`.

## Agent Configuration
[Back to Top](#pipeline-configuration)

The agent is the virtual machine that will execute the build for your project. You can create many agents assigned to a single agent pool.

In order to build your project, the service account profile needs to be setup with a subset of your development environment configuration.

> For our app stack, the following environment setup needs to be conducted:
> * git
> * node
> * PowerShell Core
> * NuGet cache
> * NuGetFallbackFolder
> * yarn / npm cache
> * .dotnet user profile folder (for dotnet-ef tools)

Once the environment configuration is complete, the virtual machine needs to be configured as a DevOps build pipeline agent.

> Ensure you have the [pipelines agent script package](#pipelines-agent-script-package) setup as outlined above.

Open an administrative PowerShell terminal pointed at the `DevOpsAgent` directory, then run `.\config.cmd`.

```pwsh
C:\DevOpsAgent> .\config.cmd

>> Connect:

Enter server URL > http://{devops-instance}.{domain}.net
Enter authentication type (press enter for Integrated) >
Connecting to server ...

>> Register Agent:

Enter agent pool (press enter for default) >
Enter agent name (press enter for {agent-name}) >
Scanning for tool capabilities.
Connecting to the server.
Successfully added the agent
Testing agent connection.
Enter work folder (press enter for _work) >
{DTG}: Settings Saved.
Enter run agent as service? (Y/N) (press enter for N) > Y
Enter User account to use for the service (press enter for NT AUTHORITY\NETWORK SERVICE) > {svc-account}
Enter password for the account {svc-account} > ************
Granting file permissions to `{svc-account}`
Service vstsagent.{devops-instance}.Default.{agent-name} successfully installed
Service vstsagent.{devops-instance}.Default.{agent-name} successfully set recovery option
Service vstsagent.{devops-instance}.Default.{agent-name} successfully set to delayed auto start
Service vstsagent.{devops-instance}.Default.{agent-name} successfully configured
Service vstsagent.{devops-instance}.Default.{agent-name} started successfully
C:\DevOpsAgent> _
```

You will need the following information:

input | example
------|--------
**Server URL** | `http://{devops-instance}.domain.net`
**Run Agent as Service** | Yes
**Service Account** | `{svc-account}`
**Service Account Password** | `***********`

If you open the *Services* tab in **Task Manager**, you should see the agent service running.

Now if you open the **Default** agent pool in Azure DevOps, then click the *Agents* tab, you should see your associated agent in the pool.

> If you ever want to remove an agent from the pool, oepn an administrative `pwsh` session at `C:\DevOpsAgent` and run `.\config remove`.

## Deployment Group Configuration
[Back to Top](#pipeline-configuration)

Although you created a build agent, it is not actually installing the app anywhere. You can think of the build agent as the continuous integration aspect of CI / CD. In order to deploy your application, you must create a deployment group with at least one deployment agent.

Similar to an agent pool, a deployment group manages your deployment agent resources. Unlike an agent pool, there are no default deployment groups.

Click **Deployment groups** in the **Pipelines** tab of your project.

Click **Add a deployment group**, provide a name, and click **Create**.

Now that the deployment group is created, you now have access to the deployment agent configuration script (which was used to discern how to create the [pipelines agent script package](#pipelines-agent-script-package)).

**Example Deployment Agent Configuration Script**

```pwsh
$ErrorActionPreference="Stop";

If(-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator"))
{
    throw "Run command in an administrator PowerShell prompt"
};

If($PSVersionTable.PSVersion -lt (New-Object System.Version("3.0")))
{
    throw "The minimum version of Windows PowerShell that is required by the script (3.0) does not match the currently running version of Windows PowerShell."
};

If (-NOT (Test-Path $env:SystemDrive\'azagent'))
{
    mkdir $env:SystemDrive\'azagent'
};

cd $env:SystemDrive\'azagent';

for($i=1; $i -lt 100; $i++)
{
    $destFolder="A"+$i.ToString();

    if (-NOT (Test-Path ($destFolder)))
    {
        mkdir $destFolder;
        cd $destFolder;
        break;
    }
};

$agentZip="$PWD\agent.zip";
$DefaultProxy=[System.Net.WebRequest]::DefaultWebProx;

$securityProtocol=@();
$securityProtocol+=[Net.ServicePointManager]::SecurityProtocol;
$securityProtocol+=[Net.SecurityProtocolType]::Tls12;

[Net.ServicePointManager]::SecurityProtocol=$securityProtocol;

$WebClient=New-Object Net.WebClient;
$Uri='https://vstsagentpackage.azureedge.net/agent/2.153.1/vsts-agent-win-x64-2.153.1.zip';

if ($DefaultProxy -and (-not $DefaultProxy.IsBypassed($Uri)))
{
    $WebClient.Proxy = New-Object Net.WebProxy($DefaultProxy.GetProxy($Uri).OriginalString, $True);
};

$WebClient.DownloadFile($Uri, $agentZip);

Add-Type -AssemblyName System.IO.Compression.FileSystem;
[System.IO.Compression.ZipFile]::ExtractToDirectory($agentZip, "$PWD");

.\config.cmd --deploymentgroup --deploymentgroupname "IIS" --agent $env:COMPUTERNAME --runasservice --work '_work' --url 'http://{devops-instance}.{domain}.net/' --collectionname '{collection-name} --projectname '{project-name}' --auth Integrated;

Remove-Item $agentZip;
```

## Deployment Agent Configuration
[Back to Top](#pipeline-configuration)

The following steps will demonstrate how to setup an IIS virtual machine as a deployment agent target for the release pipeline in a deployment group.

1. Open **Computer Management** and add your service account to the local **Administrators** group.
2. Install **IIS** from **Server Manager** (be sure to include all recommended features when prompted) by clicking *Add Roles and Features* wtih the following options:
    * Server Roles
        * Web Server (IIS)
    * Features
        * WinRM IIS Extension
    * Role Services
        * Security > Windows Authentication
        * Application Development > WebSocket Protocol
        * Management Tools > Management Service
3. Open **IIS Manager**, expand **Sites**, and remove the **Default Web Site**.
4. Install [vc_redist.x64.exe](https://aka.ms/vs/16/release/vc_redist.x64.exe). 
    * acquired from [Microsoft Support - Latest Visual C++ Downloads - VS 2015, 2017, and 2019](https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads)
5. Install [dotnet-hosting-3.1.9-win.exe](https://dotnet.microsoft.com/download/dotnet-core/thank-you/runtime-aspnetcore-3.1.9-windows-hosting-bundle-installer).
    * acquired from [dotnet-core 3.1 downloads](https://dotnet.microsoft.com/download/dotnet-core/3.1) under <span>ASP.NET</span> Core Runtime on the right via the `Hosting Bundle` link.
6. Restart the virtual machine.
7. Set **Service Account** full control directory permissions to:
    * `C:\Windows\System32\inetsrv` - With inheritance to child objects and containers.
    * `%ALLUSERSPROFILE%\Microsoft\Crypto\RSA\MachineKeys` - With inheritance to child objects and containers.
8. Install the DevOps Agent
    1. Open an Administrative PowerShell session pointed at your `DevOpsAgent` directory and run the `create-dep-agent.cmd` script.
    2. No deployment group tags needed.
    3. Enter the fully qualified **Service Account** name. Example: `service@domain.net`
    4. Provide the **Service Account** password.
    8. Verify the service is running in `Services.msc`.

You can verify this deployment agent is successfully configured by viewing the **Targets** in the **Deployment group**.

## Build Pipeline Setup
[Back to Top](#pipeline-configuration)

The **Build Pipeline** for your project is directly configured by an `azure-pipelines.yml` file in the root of your project. Here is an example:

```yml
# ASP.NET Core
# Build and test ASP.NET Core projects targeting .NET Core
# Add steps that run tests, create a NuGet pacakge, deploy, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/dotnet-core

trigger:
-master

pool:
    name: 'Default'

variables:
    buildConfiguration: 'Release'
    connection: 'Server={server};Database={database};Trusted_Connection=True;'

steps:
    - script: dotnet run -- $(connection)
      displayName: 'Seed Database'
      workingDirectory: 'dbseeder/'
    - task: DotNetCoreCLI@2
      displayName: 'Build Release'
      inputs:
        command: publish
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    - task: PublishBuildArtifact@1
      displayName: 'Publish Artifacts'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: '{artifact-name}'
```

The build configuration contains three steps:

1. Execute the **dbseeder**
2. Publish thte project in **Release** configuration
3. Publish the **Artifacts** to the build pipeline

> If your project does not have an `azure-pipelines.yml` file, one will be generated when you create an initial build pipeline.

## New Build Pipeline
[Back to Top](#pipeline-configuration)

1. Open the **Pipelines > Builds** link in your project.
2. Click the **New Pipeline** button.
3. Select **Azure Repos Git**.
4. Select your project.
5. Configure your build pipeline if needed.
    * This is where the configuration file is generated in your project if one does not exist.
6. Click **Run**.
    * This will kickoff your first build.

> IF you want to setup your **Release pipeline** directly from the build, click the **Release** button at the top of the build summary. After explaining the IIS Extensions in the next section, the **Release pipeline** configuration will be generated this way.

## Release Pipeline Setup
[Back to Top](#pipeline-configuration)

> Ensure the Azure DevOps Extension for IIS Deployment is installed:  
Downloaded from: https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.iiswebapp  
From the Collection Dashboard: Collection Settings > Extensions > IIS Web App Deployment Using WinRM  

1. After clicking **Release** from your build summary, select the **IIS website deployment** template and click **Apply**.
    * The reason the pipeline is generated from the build is because it links the build artifacts to the release pipeline.
2. Provide a **Stage** name (i.e. Deploy), then close the stage dialog.
3. Click the **Continuous deployment trigger** icon on your project name in the **Artifacts** section.
4. Click the **Add** button, select the **master** branch, then close the dialog.
5. Click the **Tasks** tab.
6. In the stage configuration (in this case, Deploy), set the appropriate website name.
7. Click the **IIS Deployment** section. Select your Deployment group from the drop-down.
8. Click the **IIS Web App Manage** section and perform the following:
    1. Check *Create or update app pool*.
    2. Check *Configure authentication*.
    3. In the **IIS Application pool** section:
        1. Provide the app pool name (i.e. **DefaultAppPool**).
        2. Set .NET version to: **No Managed Code**.
        3. Change the identity to: **Custom Account**.
        4. Specify the fully qualified **Service Account** name in the Username section (i.e. `service@domain.net`).
        5. Provide the **Service Account** password in the Password section.
    4. In the **IIS Authentication** section, ensure only **Windows Authentication** is checked.
    5. Click **Save**, then click **OK** in the proceding dialog.
9. Click the **Pipelines > Releases** link, then click **Create a release**.
10. In the **Create a new release** dialog, click **Create**.

Now that your pipelines are configured, any time your `main` branch is updated, a build will be generated, and it will be deployed to any target in your deployment group.

You can verify the deployment by navigating to the IP of your virtual machine hosting the app.
