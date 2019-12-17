# Deploy to a Windows VM

* [Prerequisites](#prerequisites)
* [Create a Deployment Group](#create-a-deployment-group)
* [Define Your CD Release Pipeline](#define-your-cd-release-pipeline)
* [Create a Release to Deploy Your App](#create-a-release-to-deploy-your-app)

We'll show you how to set up continuous deployment of your <span>ASP.NET</span> or Node.js app to an IIS web server running on Windows using Azure Pipelines.

> You'll need to build a continuous integration (CI) build pipeline that publishes your web deployment package. See [.NET Core Pipelines Ecosystemy](04-dotnet-core.md).  

## Prerequisites
[Back to Top](#deploy-to-a-windows-vm)  

### IIS Configuration

The configuration varies depending on the type of app you are deploying.

<strong><span>ASP.NET</span> Core app</strong>  

Running an <span>ASP.NET</span> Core app on Windows requires some dependencies.

Open your VM, open an **Administrator: Windows PowerShell** console. Install IIS and the required .NET features:

```ps
# Install IIS
Install-WindowsFeature Web-Server,Web-Asp-Net45,Net-Framework-Features

# Install the .NET Core SDK
Invoke-WebREquest https://go.microsoft.com/fwlink/?linkid=848827 -outfile $env:temp\dotnet-dev-win-x64.1.0.6.exe
Start-Process $env:temp\dotnet-dev-win-x64.1.0.6.exe -ArgumentList '/quiet' -Wait

# Install the .NET Core Windows Server Hosting bundle
Invoke-WebRequest https://go.microsoft.com/fwlink/?linkid=817246 -outfile $env:temp\DotNetCore.WindowsHosting.exe
Start-Process $env:temp\DotNetCore.WindowsHosting.exe -ArgumentList '/quiet' -Wait

# Restart the web server so that system PATH updates take effect
Stop-Service was -Force
Start-Service w3svc
```

## Create a Deployment Group
[Back to Top](#deploy-to-a-windows-vm)  

Deployment groups in Azure Pipelines make it easier to organize the servers that you want to use to host your app. A deployment group is a collection of machines with an Azure Pipelines agent on each of them. Each machine interacts with Azure Pipelines to coordinate deployment of your app.

1. Open the Azure Pipelines web portal and choose **Deployment Groups**  
2. Click **Add Deployment group** (or **New** if there are already deployment groups in place).
3. Enter a name for the group, such as *myIIS*, and then click **Create**.
4. In the **Register machine** section, make sure that **Windows** is selected. A script will be available for configuring a deployment group agent.
    * You cannot use this step on the inside because there's no way to download the agent onto the VM. Instead, the agent package is stored on the share drive, and you will need to configure it manually based on the specified parameters in the provided deployment group script.
5. Run the configuration using the parameters specified in the deployment group agent configuration script.
6. When you're prompted to configure tags for the agent, press Enter (you don't need any tags).
7. When you're prompted for the user account, press Enter to accept the defaults.
> The account under which the agent runs needs **Manage** permissions for the `C:\Windows\system32\inetsrv\` directory. Adding non-admin users to this directory is not recommended. In addition, if you have a custom user identity for application pools, the identity needs permission to read the crypto-keys. Local service accounts and user accounts must be given read access for this. For more details, see [Keyset does not exist error message](https://support.microsoft.com/help/977754/-keyset-does-not-exist-error-message-when-you-try-to-change-the-identi).

> Resolution for `Keyset does not exist error message`:
>
> 1. Locate the following folder: `%ALLUSERSPROFILE%\Microsoft\Crypto\RSA\MachineKeys
> 2. Right-click the following file, and then click **Properties**: `76944fb33636aeddb9590521c2e8815a_GUID`
> 3. Click the **Security** tab, and then click **Edit**. If you are asked whether you want to continue the operation, click **Continue**. Then, the list of group names and user names that have access to this key file appears in the **Permissions** dialog box.
> 4. Click **Add**. Then, the **Select Users, Computers, Service Accounts, or Groups** dialog box appears.
> 5. Type *Local Service* then click **Check Names**.
> 6. Click **OK**.
> 7. In the **Group or user names** list, click **Local Service**. Make sure that the **Read** check box is checked in the **Permissions for LOCAL SERVICE** list.
> 8. Click **OK**.  

8. When configuration is complete, it displays the message *Service vstsagent.account.computername started successfully*.
9. On the **Deployment groups** page in Azure Pipelines, open the *myIIS* deployment group. On the **Targets** tab, verify that your VM is listed.

## Define Your CD Release Pipeline
[Back to Top](#deploy-to-a-windows-vm)  

Your CD release pipeline picks up the artifacts published by your CI build and then deploys them to your IIS servers.

1. If you haven't already done so, install the [IIS Web App Deployment Using WinRM](https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.iiswebapp) extension from Marketplace. This extension contains the tasks required for this example.
2. Do one of the following:
    * If you've just copmleted a CI build then, in the build's **Summary** tab choose **Release**. This creates a new release pipeline that's automatically linked to the build pipeline.

    * Open the **Releases** tab of **Azure Pipelines**, opent the **+** drop-down in the list of release pipelines, and choose **Create release pipeline**.
3. Select the **IIS Website Deployment** template and choose **Apply**.
4. If you created your new release pipeline from a build summary, check that the build pipeline and artifact is shown in the **Artifacts** section of the **Pipeline** tab. If you created a new release pipeline from the **Releases** tab, choose the **+ Add** link and select your build artifact.
5. Choose the **Continuous deployment** icon in the **Artifacts** section, check that the continuous deployment trigger is enabled, and add a filter to include the **master** branch.
6. Open the **Tasks** tab and select the **IIS Deployment** job. For the **Deployment Group**, select the deployment group you created earlier (such as *myIIS*).
7. Save the release pipeline.

## Create a Release to Deploy Your App
[Back to Top](#deploy-to-a-windows-vm)  

You're now ready to create a release, which means to run the release pipeline with the artifacts produced by a specific build. This will result in deploying the build:

1. Choose **+ Release** and select **Create a release**.
2. In the **Create a new release** panel, check that the artifact version you want to use is selected and choose **Create**.
3. Choose the release link in the information bar message. For example: "Release **Release-1** has been created".
4. In the pipeline view, choose the status link in the stages of the pipeline to see the logs and agent output.
5. After the release is complete, navigate to your app and verify its contents.