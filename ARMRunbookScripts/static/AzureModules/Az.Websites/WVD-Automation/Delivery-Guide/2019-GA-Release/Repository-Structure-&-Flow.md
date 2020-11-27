> This site describes how the provided repository is set up and how the deployment works from a pipeline-orchestration perspective. 

[[_TOC_]]

# Repository Structure

> ![svg-azure-repos.svg](/.attachments/svg-azure-repos-0ab2955c-dd76-4674-a6b4-a553c0be94d5.svg =15x)

To undestand the deployment let's first take a look on the file structure:
| | | |         | Description | 
| ------------ | --------------------- | --------- | ---------- | ----------- |
| Environments |                       |           |            | Parent folder for all pipeline related content | 
|              | _Shared-Functions     |           |            | Contains are scripts used by the module-orchestrators in `'.\pipeline-orchestrated\wvdShared\scripts'` |
|              | _Shared-Variables     |           |            | Contains variables all pipelines should share. Good candidates are e.g. the service connection name or specific resource group names |
|              | pipeline-orchestrated |           |            | Parent folder for all pipeline-orchestrated setups |
|              |                       | wvdShared |            | Our WVD pipeline folder. Contains all required resources for this approach and the orchestration pipeline `'custom.pipeline.yml'`|
|              |                       |           | parameters | All parameters used by the different deployments jobs + shared variables for the scripts |
|              |                       |           | resources  | All pipeline templates (YAML) to be referenced by the orchestration pipeline. Includes main deployment template plus all pre- and post-deployment pipeline templates |
|              |                       |           | scripts    | Post- and pre-deployment orchestration files. All scripts in here reference logic specified in the '__Shared-Functions_' folder |
|              |                       |           | `custom.pipeline.yml`| The deployment / orchestration pipeline to control all other parts |
| Modules      |                       |           |                          | Contains the collection of modules used in this deployment. The content of this folder can be used to further iterate on the ARM templates and test any changes. For further details look [here](https://dev.azure.com/SecInfra/Components/_wiki/wikis/Start%20Here/644/DevOps-Enablement?anchor=(optional)-set-up-the-build-pipeline-for-each-module). |
| WVDADAuthPrereq |                       |           |                          | A PowerShell module providing a number of helper scripts to be executed e.g. to securely store the domain join user credentials in a key vault secret, or to enable Azure Files AD DS authentication (preview). <br/> **Note**: This module is only used within the _**Native AD approach**_. |   
|              | Private               |           |                          | Contains all helper functions for the public functions
|              | Public                |           |                          | Contains all exported functions of this module. **Note:** Once the module is imported, the functions must be called with the default command prefix (i.e. module name)
|              | Static                |           |                          | Contains the _AzFilesHybrid_ module (currently v0.1.1.0) which is imported in the PSModulePath prior to enable the  Azure Files AD DS authentication (preview) feature.|
|              | Test                  |           |                          | Contains test functions for the current module to ensure code quality and functionality |
|              |                       | ModuleConfig.psd1 |                  | A configuration file for centralized static content to be referenced by the modules functions. Loaded when module is important and is accessible via `Script:ModuleConfig.*` |
|              |                       | WVDADAuthPrereq.psd1    |                  | The module's manifest with its configuration, e.g. the default command prefix and required modules. |
|              |                       | WVDADAuthPrereq.psm1    |                  | The module's module file that loads the configuration, all functions and exports those that in the public folder |
| WVDPreReq    |                       |           |                          | A PowerShell module providing a number of helper scripts to e.g. set-up the WVD tenant, or generate parameter files for the deployment. |   
|              | Private               |           |                          | Contains all helper functions for the public functions
|              | Public                |           |                          | Contains all exported functions of this module. **Note:** Once the module is imported, the functions must be called with the default command prefix (i.e. module name)
|              | Static                |           |                          | Contains the module configuration files. Contains files such as parameter templates, configuration files and the custom WVD schema |
|              | Test                  |           |                          | Contains test functions for the current module to ensure code quality and functionality |
|              |                       | ModuleConfig.psd1 |                  | A configuration file for centralized static content to be referenced by the modules functions. Loaded when module is important and is accessible via `Script:ModuleConfig.*` |
|              |                       | WVDPreReq.psd1    |                  | The module's manifest with its configuration like the default command prefix, required modules, etc. |
|              |                       | WVDPreReq.psm1    |                  | The module's module file that loads the configuration, all functions and exports those that in the public folder |
# Execution flow

> ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =15x)

To run the deployment pipeline you can use the provided YAML pipelines. In each phase of the pipeline, a parameter file is used and must hence be configured prior to the pipeline's execution. To understand how this works, let's have a look at the generic pipeline structure:


![pipelineSchema.png](/.attachments/pipelineSchema-d520e494-0a71-4958-8531-9478918d10c6.png =90%x)

# Elements
The following is an example for a `custom.pipeline.yml` file.
```YAML
name: WVD deployment

variables:
- template: '../../_Shared-Variables/variables.shared.yml'
- template: './parameters/variables.shared.modules.yml'

trigger: none

stages:
- stage: SBX
  jobs:
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn: ""
      environment: SBX
      moduleName: KeyVault
      moduleVersion: '2019-11-28'
      parameterFilePath: 'Environments/pipeline-orchestrated/wvdShared/parameters/keyvault.parameters.json'
      preDeploymentPipelinePath: ''
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      postDeploymentPipelinePath: ./pipeline.steps.keyVault.postDeployment.yml
      resourcegroup: '$(base-rg)'
      isEnabled: true

  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn: ""
      environment: SBX
      moduleName: StorageAccounts
      moduleVersion: '2019-04-01'
      parameterFilePath: 'Environments/pipeline-orchestrated/wvdShared/parameters/storageaccount.parameters.json'
      preDeploymentPipelinePath: ''
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      postDeploymentPipelinePath: ./pipeline.steps.storageAccount.postDeployment.yml
      resourcegroup: '$(base-rg)'
      isEnabled: true

  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_KeyVault
      - DeployModule_StorageAccounts
      environment: SBX
      moduleName: WVD
      moduleVersion: '2019-12-25'
      parameterFilePath: 'Environments/pipeline-orchestrated/wvdShared/parameters/wvd.parameters.json'
      preDeploymentPipelinePath: ''
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      postDeploymentPipelinePath: './pipeline.steps.wvd.postDeployment.yml'
      resourcegroup: '$(pool-rg)'
      isEnabled: true
```
Let's take a look, what each element represents and how you can configure it.
## Root / Orchestrator File 
The main orchestrator is the `custom.pipeline.yml` file. It contains all references to **variable** and **job** templates and contains the configuration for the deployment

## Variables
By default the `custom.pipeline.yml` loads two variable files:
- `'variables.shared.yml'` provides basic variables such as the service connection name, resource group names, etc.
- `'variables.shared.modules.yml'` contains information that is already available in the different `parameters.json` files, but is required in the post- and pre-deployment scripts.

## Stages
The `custom.pipeline.yml` can contain one or multiple stages. By default, a Sandbox stage (SBX) is provided.  Each stage references deployment **job** templates

### Jobs

Each **job** template offers you three deployment templates:
  - Pre-Deployment (via YAML reference) 
  - Deployment (via YAML reference)
  - Post-Deployment (via YAML reference)

Depending on whether you use one or multiple of these deployment templates, you must provide a set of parameters:
- Pre-Deployment
  | Parameter  | Description                                                                    |
  |------------|--------------------------------------------------------------------------------|
  | preDeploymentPipelinePath | Path to the pre-deployment YAML.<p><p>E.g. `'preDeploymentPipelinePath: ./pipeline.steps.storageAccount.preDeployment.yml'` |
  | moduleName | The name of the module that this template belongs to.<p><p>E.g. `'moduleName: StorageAccount'`  |

- Deployment
  | Parameter  | Description                                                                    |
  |------------|--------------------------------------------------------------------------------|
  | deploymentPipelinePath | Path to the standard deployment YAML.<p><p>E.g. `'././pipeline.steps.deployment.yml'` |
  | moduleName | The name of the module that this template belongs to.<p><p>E.g. `'moduleName: StorageAccount'`  |
  | moduleVersion | Module version pushed to the template storage account. Is needed to build the path to the ARM template and can be found in the template storage accounts path to the modules ARM template (reference: `'https://<storageAccountName>.blob.core.windows.net/templates/Modules/ARM/<moduleName>/<moduleVersion>/deploy.json'`). <p><p>E.g. `'moduleVersion: 2019-11-28'`|
  | resourcegroup | Name of the resource group to deploy into. (as specified in the `'variables.shared.yml'`).<p><p>E.g. `'$(base-rg)'` |
  | parameterFilePath | Path to the modules parameter file used in the ARM deployment.<p><p>E.g. `''Environments/pipeline-orchestrated/wvdShared/parameters/storageaccount.parameters.json''` |

- Post-Deployment
  | Parameter  | Description                                                                    |
  |------------|--------------------------------------------------------------------------------|
  | postDeploymentPipelinePath | Path to the post-deployment YAML. <p><p>E.g. `'postDeploymentPipelinePath: ./pipeline.steps.storageAccount.postDeployment.yml'` |
  | moduleName | The name of the module that this template belongs to. <p><p>E.g. `'moduleName: StorageAccount'`  |
 
- Optional Parmeters
  | Parameter  | Description                                                                    |
  |------------|--------------------------------------------------------------------------------|
  | dependsOn  | Defines dependencies between the specified deployment jobs (as indicated via the dotted arrow to the '...' job in the diagram).<p><p>E.g. `'dependsOn: - DeployModule_KeyVault'` |
  | isEnabled  | Enables control whether this job is enabled for the next deployment or not.<p><p>E.g. `'isEnabled: true'` |
  | suffix     | A suffix to add to the job's name in the pipeline. This parameter is required when deploying multiple jobs for the same component. As each job requires a unique name, you can use this parameter to differentiate each job from one another.<p><p>E.g. `'suffix: Primary'` |
  | serviceConnection | The name of the service connection to use for this job. Defaults to the variable "serviceConnection" specified in `'variables.shared.yml'`<p><p>E.g. `'serviceConnection: WVDDeploymentServicePrincipalConnection'` |
