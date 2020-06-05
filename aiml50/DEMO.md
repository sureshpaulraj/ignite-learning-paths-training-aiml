# AIML50 - Demonstration Setup Instructions

## Create Demonstration Environment

[Video Walkthrough](https://youtu.be/C9WtOZaUoyA)

### Prerequisites

* An Azure subscription
* An Azure DevOps organization that you have rights to add extensions to.
  * A Personal Access Token(PAT) for that organization.
* A GitHub account (to which you can fork this repository)

### Fork the repository

In GitHub, [create a fork](https://help.github.com/en/github/getting-started-with-github/fork-a-repo) of this repository under a user or organization of which you have control.  You will need permissions to connect the GitHub repo to Azure DevOps.

### Deploy the Template

This environment can be deployed via the "Deploy to Azure" link below (or you can use Azure PowerShell or Azure CLI).  You will need an Azure subscription and the available quotas in a region to deploy:

* Azure SQL Databases
* Cosmos DB Databases
* Azure App Services
* Azure Machine Learning Services

You will be prompted to select an Azure subscription and resource group (you can create a resource group at that time).

You will also be asked for an event identifier (or reason for spinning up the environment) which will be used to help name the resources.  Shorter is better.

You will need to provide a database username and password for the Azure SQL instance.

*Not all resources are available in all regions. For Europe choose: North Europe*

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3a%2f%2fraw.githubusercontent.com%2fmicrosoft%2fignite-learning-paths-training-aiml%2fmaster%2faiml50%2ftemplate%2fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

The deployment takes roughly 30 to 40 minutes.

Once the deployment is underway (at least with the Azure Machine Learning service created and the bootstrap-container Azure Container instance has run to completion), you can finish setting up the Azure DevOps environment.  Most of the environment will be configured, but there are a few manual steps.

### Set up Azure Notebooks

* Navigate to [Azure Notebooks](https://notebooks.azure.com/) and sign in with the Microsoft account that you are demoing with.
* Add a new project.  You can either import directly from GitHub (the main repository or your fork) or upload the `aiml50/source` directory directly.
* In the `aiml50/source` directory in the Azure Notebook, create a json file named `azureml-config.json` with:
  * Your subscription ID
  * The resource group name that contains the ML workspace
  * The workspace name

Example:

```
{
    "subscription_id": "cd400f31-6f94-40ab-863a-673192a3c0d0",
    "resource_group": "aiml50",
    "workspace_name": "aiml50demo"
}
```


* Also, add `location` parameter to your region in the `deploymentconfig.json` file if necessary. The default ACI region is westus.

Example:
```
{
   "containerResourceRequirements": {
       "cpu":        2,
       "memoryInGB": 4
    },
    "computeType":       "ACI",
    "enableAppInsights": "True",
    "location":          "westus"
}
```
Reference: https://docs.microsoft.com/en-us/azure/container-instances/container-instances-region-availability

* Click on (which will open in a new tab)
  * `seer_pipeline.ipynb`

#### seer_pipeline.ipynb

* Ensure the kernel is set to Python 3.6
* Set your storage account key
* edit Step 4 and set your storage account name
* Start to run the individual steps.  You will need to authenticate to azure (follow the prompts in the notebook). Remember to let individual steps finish before starting the next one.

### Setup the Azure DevOps Project

Next, navigate to the AIML50 project that was created in the Azure DevOps Organization you specified to to the deployment template.

#### Create the Build

Now, we need to create a build definition by pointing Azure DevOps to our build definition on GitHub.

* Navigate to `Pipelines` (under Pipelines).
* Select `New Pipeline`

![9-azure_devops_pipeline_new](./images/9-azure_devops_pipeline_new.png)
![10-azure_devops_pipeline_new_source](./images/10-azure_devops_pipeline_new_source.png)

* Connect to your fork of the GitHub project [Ignite Learning Paths Training AI/ML](https://github.com/microsoft/ignite-learning-paths-training-aiml)

![11-azure_devops_pipeline_select_repo](./images/11-azure_devops_pipeline_select_repo.png)

![12-azure_devops_pipeline_select_build_definition](./images/12-azure_devops_pipeline_select_build_definition.png)

* Choose to use the build definition from the repository (`aiml50/azure-pipelines.yml`)

![13-azure_devops_pipeline_select_build_definition_location](./images/13-azure_devops_pipeline_select_build_definition_location.png)

#### Run the Build
make sure to verify your Azure DevOps org's PAT value for variable "access_token" under variable group name "aiml50-demo"
After the build is connected to the source repository, we need to run a build to create the Machine Learning pipeline and create a build artifact so we can finish setting up the release pipeline.

* Review the build definition and run the build. The build will complete in a few minutes, but it triggers a Machine Learning pipeline which can take about 20-40 minutes.

![14-azure_devops_pipeline_review_build_definition](./images/14-azure_devops_pipeline_review_build_definition.png)
![15-azure_devops_pipeline_build_result](./images/15-azure_devops_pipeline_build_result.png)

#### Update the Release

After the Machine Learning pipeline finishes, we can update the release pipeline.

* Navigate to `Releases` (under Pipelines).

![16-azure_devops_release_new](./images/16-azure_devops_release_new.png)

* Select `Release Seer` and choose `Edit`

![17-azure_devops_release_edit](./images/17-azure_devops_release_edit.png)

  * Select `Add an artifact`
![18-azure_devops_release_artifact](./images/18-azure_devops_release_artifact.png)


  * Set a `Source type` of `AzureML`
  * Set the service endpoint to `aiml50-workspace`
  * Set the Model Names to `seer`.  You will not be able to do this until the first ML Pipeline finishes.
  * Click `Add`
  * Click the lightning icon on the new artifact and enable the `Continuous deployment trigger`

![19-azure_devops_release_artifact_set](./images/19-azure_devops_release_artifact_set.png)


* Next, open the `Deploy to ACI` environment.

![20-azure_devops_release_edit_2](./images/20-azure_devops_release_edit_2.png)

* Click on `Agent Job`
  * Set `Agent Pool` to `Azure Pipelines`
  * Set `Agent Specification` to `ubuntu-18.04`

![21-azure_devops_release_task_agent](./images/21-azure_devops_release_task_agent.png)

* Click on `Download deployment and inferencing code`
  * Set `Package name` to `seer_deployment`

![22-azure_devops_release_task_edit](./images/22-azure_devops_release_task_edit.png)

* Click on `Azure ML Model Deploy`
  * Verify that Azure ML Workspace is set to either `$(subscription_workspace)` or `aiml-workspace`.

![23-azure_devops_release_task_verify](./images/23-azure_devops_release_task_verify.png)

* Save the pipeline and create a new release.



## Troubleshooting and Reference

### Checking the container deployment log

In the provisioned resource group, navigate to the `bootstrap-container` container instance. From there, you can check the logs for the container, which will show the steps taken and any errors encountered.

After deployment model to ACI, please check all **3** container run. If it's terminated, please **restart** ACI instance.

### Provider registration

The Tailwind Traders application uses many Azure services. In some cases, if a service has not yet been used in your subscription, a provider registration may be needed. The following commands will ensure your subscription is capable of running the Tailwind Traders application.

```
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.DocumentDB
az provider register --namespace Microsoft.DBforPostgreSQL
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.ContainerRegistry
```

### Source Repositories

https://github.com/microsoft/TailwindTraders

https://github.com/microsoft/TailwindTraders-Backend

https://github.com/microsoft/TailwindTraders-Website
