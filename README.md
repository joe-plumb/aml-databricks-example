# Template for ML workloads using Azure Machine Learning and Azure Databricks
This repo contains code and instructions for standing up an example project leveraging best practices for Machine Learning pipelines using Azure Machine Learning, Azure Databricks, and Azure Blob Storage. The notebooks and code here can be used as a reference and lab to understand the process of moving from an "all-in-one" end to end ML process in one notebook, to breaking that down into separate, distinct steps (i.e. data prep, training, evaluation), tieing those steps together using ML pipelines, and parameterizing secrets and runtime variables. Additionally, the example showcases Azure Machine Learning's AutoML. You can learn more about [how AutoML works in the documentation](https://docs.microsoft.com/en-us/azure/machine-learning/concept-automated-ml).

## Build and deploy the Template

### 1. Azure Services deployment
First, you will need to make sure you have access to the required services in your Azure Subscription. You can create new instances of these if you are exploring, or use existing resources if you are already working with these services. Ensure you have instances of:
 - [Azure Databricks](https://ms.portal.azure.com/#create/Microsoft.Databricks) - (workspace and cluster)
 - [Azure Machine Learning](https://ms.portal.azure.com/#create/Microsoft.MachineLearningServices) - (Basic SKU is sufficient)
 - [Azure Key Vault](https://ms.portal.azure.com/#create/Microsoft.KeyVault)
Deploy all into the same resource group to simplify clean up.

### 2. Azure Databricks setup
#### Create and configure your cluster
Launch your Azure Databricks workspace and [create a new interactive cluster](https://docs.microsoft.com/en-us/azure/databricks/clusters/create). For demonstration purposes, we are going to use an interactive cluster, however take note that automated (jobs) clusters are available and are more optimally priced for non-interactive workloads.
Once the cluster has been created, install the required Azure Machine Learning libraries [as detailed in the documentation](https://docs.microsoft.com/en-gb/azure/machine-learning/how-to-configure-environment#azure-databricks). This repo uses AutoML, so if you plan to deploy this workflow or wish to work with AutoML, ensure you add the `azureml-sdk[automl]` library.
#### Generate and store Databricks Personal Access Token (PAT)
Leveraging Databricks resources in pipelines requires a PAT in order to authenticate against the workspace. [Generate a PAT using the UI](https://docs.databricks.com/dev-tools/api/latest/authentication.html#generate-a-personal-access-token), or, to generate this as part of an automated deployment [you can use the CLI](https://cloudarchitected.com/2020/01/provisioning-azure-databricks-and-pat-tokens-with-terraform/).
Once you have generated the PAT, [add it to your key vault secrets](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-portal#add-a-secret-to-key-vault).
#### Databricks Secret Scope
In order to retrieve and manage Databricks Secrets from your Key Vault, you will need to create a Key Vault-backed secret scope. [Follow these instructions](https://docs.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes#--create-an-azure-key-vault-backed-secret-scope) to configure this, ensuring to link to the Key Vault you created as part of step 1 (not the Key Vault that was provisioned as part of the Azure Machine Learning deployment). Name the secret scope `databricks-aml-demo`.

### 3. Service principal setup
#### 1. Create a service principal
An Azure service principal is a security identity used by user-created apps, services, and automation tools to access specific Azure resources. Think of it as a 'user identity' (username and password or certificate) with a specific role, and tightly controlled permissions. In this case, we will be using the Service Principal to _build and execute a Machine Learning Pipeline_. First, you will need to [create a service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#create-an-azure-active-directory-application). Note your Active Directory Tenant ID, Subscription Key, and Object Id. Create a new Client Secret called `databricks-aml-demo` and note this too. [Add the below secrets to your key vault secrets](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-portal#add-a-secret-to-key-vault), which will leave you with the following:

| Key name                           | Details                                                                          |
| ---------------------------------- | -------------------------------------------------------------------------------- |
| azure-subscription-id              | Your Azure subscription ID                                                       |
| azure-tenant-id                    | Your Azure tenant ID                                                             |
| blob-account-key                   | The access key for the default blob storage associated with your AML Workspace   |
| blob-accountname                   | The name of the default blob storage associated with your AML Workspace          |
| databricks-aml-demo-sp-client-id   | The Object Id of your Service Principal                                          |
| databricks-aml-demo-sp-client-key  | The Client Secret of your Service Principal                                      |
| databricks-pat                     | The Personal Access Token for the Databricks Workspace.                          |

#### 2. Assign RBAC roles to Service Principal
Add the following Role Assignments for the following resources to your service principal:
| Resource Type | Role |
| ------------- | ---- |
| Azure Machine Learning | Contributor |
| Azure Key Vault | `Get` and `List` permissions for Secrets |

### 3. Configure Azure Machine Learning
#### 1. Add Azure Databricks as a Compute Target
Add your Azure Databricks resource as "attached compute" in your Azure Machine Learning workspace. Navigate to https://ml.azure.com/ and select your workspace. Click `Compute` on the left hand side, then `Attached Compute`, and `+ New`. Give it a name e.g. `databricksprod`, select your Databricks Workspace, and provide the PAT you generated and stored in your Key Vault.
#### 2. Register the dataset for your pipeline
The AML Pipeline will expect the sample dataset to be uploaded and registered in your AML workspace. You can do this by uploading and running `0_configuration_blob.ipynb` in your Databricks cluster.

### 4. Set Up Azure DevOps
#### 1. Create a project
[This article describes how to use Azure DevOps to create a project](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops&tabs=preview-page#create-a-project) and establish a repository for source code.
#### 2. Import the repo to Azure DevOps
[This guide shows you how to import an existing Git repo from GitHub, Bitbucket, GitLab, or other location into a new or empty existing repo in your project in Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/repos/git/import-git-repository?view=azure-devops). You can import this repo and use it as is, or fork this repo into your own account and reference this.
#### 3. Import the build pipeline
In your Azure DevOps project, navigate to  _Pipelines_. Click _New Pipeline_ in the top right. Choose the git source you selected in the previous step (Azure Repos or GitHub). Select the repo, then _Existing Azure Pipelines YAML file_, and set the path to `azure-pipelines.yml`. Change the pipeline variables where necessary (e.g. `aml.workspace`, `aml.rg`, `databricks.host`, [`databricks.cluster.id`](https://docs.databricks.com/workspace/workspace-details.html#cluster-url-and-id)) 
#### 4. Set pipeline environment variables
Add the following variables to the pipeline:
 - `databricks.token` - Databricks PAT
 - `sp.appid` - Service Principal Object Id
 - `sp.password` - Service Principal Client Secret
 - `sp.tenant` - Azure Tenant Id
For further details on this, [see the documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-variables-in-pipeline)
TODO - Collect these secrets from Azure Key Vault rather than duplicating secret storage

### 5. Run the pipeline
You should now be ready to run the pipeline. Navigate to the pipeline and click _Run Pipeline_. Once the pipeline has successfully completed, you will be able to explore the assets that were built in your Azure ML workspace. 

## Asset Details
### Demonstration Notebooks
`0_configuration_blob.ipynb`, `1_basic_clean.ipynb`, and `2_basic_train_automl.ipynb` are the steps that make up the ML pipeline. These are stored in `/notebooks` directory in the repo.

### azure-pipelines.yml
A pipeline is one or more stages that describe a CI/CD process. Stages are the major divisions in a pipeline. The stages "Build this app," "Run these tests," and "Deploy to preproduction" are good examples. The pipeline in this repo deploys the sample notebooks to the configured Azure Databricks workspace, then builds and submits the AzureML pipeline.
You can [learn more about the `azure-pipelines.yml` specification here](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema)

### aml-pipeline.yml
The Azure ML pipeline defines the units of work to be undertaken as part of the Machine Learning workflow. This is defined in `aml-pipeline.yml`. Take a ok at the file, you will see it references the 2 python scripts that make up the ML pipeline (data prep, then training). This pipeline declaration is passed in to the `az ml` cli in the last step of the Azure DevOps build pipeline.

## Future developments
Further developments to this repo could include
- Example release pipeline with ACI and AKS
- Automated testing in release pipeline
- Deeper integration with Key Vault (AzDO)
- IaC for initial deployment
- Script Service Principal setup

## Credits
az cli and Azure Databricks integration leverages code from https://github.com/SaschaDittmann/MLOps-Databricks
Example ML code from https://docs.microsoft.com/en-us/azure/databricks/_static/notebooks/getting-started/popvspricelr.html and 
https://docs.microsoft.com/en-us/azure/databricks/getting-started/spark/machine-learning, using Databricks Sample Dataset https://docs.microsoft.com/en-us/azure/databricks/data/databricks-datasets 