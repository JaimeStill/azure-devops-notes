# Customize an Azure Pipeline

* [Change the platform to build on](#change-the-platform-to-build-on)
* [Add steps](#add-steps)
* [Build across multiple platforms](#build-across-multiple-platforms)
* [Build using multiple versions](#build-using-multiple-versions)
* [Customize CI triggers](#customize-ci-triggers)
* [Customize settings](#customize-settings)

**Understand the `azure-pipelines.yml` file**  

A pipeline is defined using a YAML file in your repo. Usually, this file is named `azure-pipelines.yml` and is located at the root of your repo.

* Navigate to the **Pipelines** page in Azure Pipelines and select a pipeline.
* Select **Edit** in the context menu of the pipeline to open the YAML editor for the pipeline.

*Example Pipeline*

``` yml
trigger:
- master

pool:
  vmImage: 'Ubuntu-16.04'

steps:
  - task: Maven@3
    inputs: 
      mavenPomFile: 'pom.xml'
      mavenOptions: '-Xmx3072m'
      javaHomeOption: 'JDKVersion'
      jdkVersionOption: '1.8'
      jdkArchitectureOption: 'x64'
      publishJUnitResults: false
      testResultsFiles: '**/surefire-reports/TEST-*.xml'
      goals: 'package'
```

This pipeline runs whenever a team pushes a change to the master branch of their repo. It runs on a Microsoft-hosted Linux machine. The pipeline process has a single step, which is to run the Maven task.

## Change the platform to build on
[Back to Top](#customize-an-azure-pipeline)  

```yml
pool:
  vmImage: 'vs2017-win2016'
```

or

```yml
pool:
  vmImage: 'macos-10.13'
```

## Add steps
[Back to Top](#customize-an-azure-pipeline)  

You can add additional **scripts** or **tasks** as steps to your pipeline. A task is a pre-packaged script. You can use tasks for building, testing, publishing, or deploying your app. For Java, the Maven task we used handles testing and publishing results, however, you can use a task to publish code coverage results too.

```yml
- task: PublishCodeCoverageResults@1
  inputs:
    codeCoverageTool: "JaCoCo"
    summaryFileLocation: "$(System.DefaultWorkingDirectory)/**/site/jacoco/jacoco.xml"
    reportDirectory: "$(System.DefaultWorkingDirectory)/**/site/jacoco"
    failIfCoverageEmpty: true
```

## Build across multiple platforms
[Back to Top](#customize-an-azure-pipeline)  

You can build and test your project on multiple platforms. One way to do it is with `strategy` and `matrix`. You can use variables to conveniently put data into various parts of a pipeline.

```yml
# replace
pool:
  vmImage: 'ubuntu-16.04'
# with
strategy:
  matrix:
    linux:
      imageName: 'ubuntu-16.04'
    mac:
      imageName: 'macos-10.13'
    windows:
      imageName: 'vs2017-win2016'
    maxParallel: 3

pool:
  vmImage: $(imageName)
```

## Build using multiple versions
[Back to Top](#customize-an-azure-pipeline)  

To build a project using different versions of that language, you can use a `matrix` of versions and a variable. In this step you can either build the Java project with two different versions of Java on a single platform or run different versions of Java on different platforms.

* If you want to build on a single platform and multiple versions:

```yml
strategy:
  matrix:
    jdk10:
      jdk_version: '1.10'
    jdk11:
      jdk_version: '1.11'
  maxParallel: 2

# Update jdkVersionOption in task
jdkVersionOption: $(jdk_version)
```

* If you want to build on multiple platforms and versions:

```yml
trigger:
- master

strategy:
  matrix:
    jdk10_linux:
      imageName: 'ubuntu-16.04'
      jdk_version: '1.10'
    jdk11_windows:
      imageName: 'vs2017-win2016'
      jdk_version: '1.11'
  maxParallel: 2

pool:
  vmImage: $(imageName)

steps:
  - task: Maven@3
    inputs:
      mavenPomFile: 'pom.xml'
      mavenOptions: '-Xmx3072m'
      javaHomeOption: 'JDKVersion'
      jdkVersionOption: $(jdk_version)
      jdkArchitectureOption: 'x64'
      publishJUnitResults: true,
      testResultsFiles: '**/TEST-*.xml'
      goals: 'package'
```

## Customize CI triggers
[Back to Top](#customize-an-azure-pipeline)  

You can use a `trigger:` to specify the events when you want to run the pipeline. YAML pipelines are configured by default with a CI trigger on your default branch (which is usually master). You can setup triggers for specific branches or for pull request validation. For a pull request validation trigger, just replace the `trigger:` step with `pr:` as shown below.

If you'd like to setup triggers:

```yml
# Based around changes
trigger:
  - master
  - releases/*

# Based around pull requests
pr:
  - master
  - releases/*
```

You can specify the full name of the branch (for example, `master`) or a prefix-matching wildcard (for example, `releases/*`).

## Customize settings
[Back to Top](#customize-an-azure-pipeline)  

There are pipeline settings that you wouldn't want to manage in your YAML file. Follow these steps to view and modify these settings:

1. From your web browser, open the project for your organization in Azure DevOps and choose Pipelines / Pipelines from the navigation sidebar.
2. Select the pipeline you want to configure settings for from the list of pipelines.
3. Open the overflow menu by clicking the action button with the vertical ellipsis and select Settings.

**Processing of new run requests**  

Sometimes you'll want ot prevent new runs from starting on your pipeline.

* By default, the processing of new run requests is **Enabled**. This setting allows standard processing of all trigger types, including manual runs.
* **Paused** pipelines allow run requests to be processed, but those requests queued without actually starting. When new request processing is enabled, run processing resumes starting with the first request in the queue.
* **Disabled** pipelines prevent uses from starting new runs. All triggers are also disabled while this setting is applied.

**Other settings**  

* **YAML file path.** If you ever need to direct your pipeline to use a different YAML file, you can specify the path to that file. This setting can also be useful if you need to move/rename your YAML file.
* **Automatically link work items included in this run.** The changes associated with a given pipeline run may have work items associated with them. Select this option to link those work items to the run. When this option is selected, you'll need to specify a specific branch. Work items will only be associated with runs of that branch.
* To get notifications when your runs fail, see how to [Manage notifications for a team](https://docs.microsoft.com/en-us/azure/devops/notifications/howto-manage-team-notifications?view=azure-devops).