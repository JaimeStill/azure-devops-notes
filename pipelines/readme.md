# Pipelines

Azure Pipelines combines continuous integration (CI) and continuous delivery (CD) to constantly and consistently test and build your code and ship it to any target.

**Deployment Targets**  

Use Azure Pipelines to deploy your code to multiple targets. Targets include container registries, virtual machines, Azure services, or any on-premises or cloud target.

**Use CI and CD for your project**  

Continuous integration is used to automate tests and builds for your project. CI helps to catch bugs or issues early in the development cycle, wehn they're easier and faster to fix. Items known as artifacts are produced from CI systems. They're used by the continuous delivery release pipelines to drive automatic deployments.

Continuous delivery is used to automatically deploy and test code in multiple stages to help drive quality. Continuous integration systems produce deployable artifacts, which includes infrastructure and apps. Automated release pipelines concume these artifacts to release new version and fixes to the target of your choice.

**Continuous integration (CI)** | **Continuous delivery (CD)**
--------------------------------|-----------------------------
Increase code coverage | Automatically deploy code to production
Build faster by splitting test and build runs | Ensure deployment targets have latest code
Automatically ensure you don't ship broken code | Use tested code from CI process
Run tests continually | 

## Links

* [Use Azure Pipelines](01-use-azure-pipelines.md)
* [Key Concepts for Azure Pipelines](02-key-concepts.md)
* [Customize a Pipeline](03-customize-pipeline.md)
* [.NET Core Pipeline Ecosystem](04-dotnet-core.md)