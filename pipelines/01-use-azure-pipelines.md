# Use Azure Pipelines

You define your pipeline in a YAML file called `azure-pipelines.yml` with the rest of your app.  

* The pipeline is versioned with your code. It follows the same branching structure. You get validation of your changes through code reviews in pull requests and branch build policies.
* Every branch you use can modify the build policy by modifying the `azure-pipelines.yml` file.
* A change to the build process might cause a break or result in an unexpected outcome. Because the change is in version control with the rest of your codebase, you can more easily identify the issue.

Follow these basic steps:  

1. Configure Azure Pipelines to use your Git repo.
2. Edit your `azure-pipelines.yml` file to define your build.
3. Push your code to your version control repository. This action kicks off the default trigger to build and deploy and then monitor the results.  

Your code is not updated, built, tested, and packaged. It can be deployed to any target.  

[![pipelines-flow](../images/pipelines-flow.png)](../images/pipelines-flow.png)  