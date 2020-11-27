This example describes a multi-host pool deployment alongside its configuration and required files.

[[_TOC_]]

This examples includes
- FSLogix with ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =10x) AADDS Authentication
- 1 ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =10x) Key Vault 
- 1 ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =10x) Storage Account (with ![Icon-storage-400-Azure-Fileshare.svg](/.attachments/Icon-storage-400-Azure-Fileshare-f7871b1c-b101-4544-afd5-bb22a3a59ae9.svg =10x) File Share) 
- 2 ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =10x) User Groups (`[File Share Access Group]`)
- 4 ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =10x) Host Pools 
  - Each with the default _'Desktop Application Group'_
  - Each with a _'Remote Application Group'_
- 5 ![Icon-general-7-Resource-Groups.svg](/.attachments/Icon-general-7-Resource-Groups-2efb1d49-d6a0-464c-80d4-7c57b78cd534.svg =10x) Resource Groups
  - One for the Baseline Deployment(s) 
  - One for each Host Pool Deployment

# Deployment
This section describes what the deployment actually does and how it is set up. It starts by introducing multiple viewpoints on the target [structure](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation?wikiVersion=GBmaster&_a=edit&pagePath=%2FExamples%2FMulti%252DHostPool&pageId=740&anchor=structure) to then dive into the most important [code](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation?wikiVersion=GBmaster&_a=edit&pagePath=%2FExamples%2FMulti%252DHostPool&pageId=740&anchor=code) pieces themselves. 

## Structure
First, you let's investigate different angles with an abstract view of the:
- [Target infrastructure](#Target-Infrastructure)
- [WVD schema](#WVD-schema)
- [Pipeline structure](#Pipeline-structure)
- [Pipeline dependencies](#Pipeline-dependencies)

### Target Infrastructure

Here you find an abstract view on the resources as they are deployed in Azure and those assumed to pre-exist (provided). 

![infra.png](/.attachments/infra-e2280050-31c2-4475-8f1f-24581ffb8529.png =70%x)

---

### WVD Schema
This is a general view on the four different host pools created in our WVD tenant alongside their corresponding 'Desktop Application Groups'. In addition, we deploy a remote application group to each to allow the use of RemoteApps on each Host. 

> Beware: User can only be assigned to either the 'Desktop Application Group' or any number of 'Remote Application Groups' at the same time.

> Note: Keep in mind you can only add RemoteApps to the Schema **after** the Session Hosts are deployed as part of the pipeline deployment.

![schema.png](/.attachments/schema-60a20fe5-d2e0-412d-bb28-73c22c51d1b1.png =50%x)

---

### Pipeline structure

The pipeline structure is a version of the [pipeline overview](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation/717/Repository-Structure-Flow?anchor=execution-flow) broken down to the current example. It shows what files are references where and how the overall deployment flow works.

![pipeline.png](/.attachments/pipeline-4ccf4e15-f558-4dba-9976-22fe3137cf0f.png =80%x)

---

### Pipeline dependencies
Following you can find an illustration of the pipeline dependencies in this setup. If you have parallel jobs, the deployment will go as folows
- First, the KeyVault & Storage Account are deployed and their post-deployment executed
- Second, the first two host pools are deployed and executed alongside the second storage account post-deployment
- Third, the remaining two host pools are deployed

![dependencies.png](/.attachments/dependencies-d7a26fad-fde1-4f34-b360-5f19cf45723e.png =50%x)

---

## Code
Following you can find the main code components for the above illustrations:
- [WVD Schema Definition](#WVD-Schema-Definition)
- [Pipeline Definition](#Pipeline-Definition)

### WVD Schema Definition
The schema describes how the tenant is structured. As we want to deploy 4 host pools, we add 4 host pools to the schema. Each pool receives both the default 'Desktop Application Group' and a dedicated 'Remote Application Group'. 

```Json
{
    "wvdTenants": [
        {
            "TenantName": "myTenant",
            "AadTenantId": "49ff757b-154a-8r4d-a676-e3f967e64a2a",
            "AzureSubscriptionId": "62826c76-8r4d-46d8-a0f6-718dbdcc536c",
            "FriendlyName": "My-Wvd-Tenant-1",
            "wvdHostPool": [
                {
                    "HostPoolName": "myHostPool",
                    "ValidationEnv": false,
                    "AssignmentType": "",
                    "Description": "Ricks Pool",
                    "FriendlyName": "WVD-Host-Pool-0",
                    "Persistent": false,
                    "wvdAppGroup": [
                        {
                            "AppGroupName": "Desktop Application Group",
                            "Description": "This is the default desktop App group",
                            "FriendlyName": "Desktop-App-Group"
                        },
                        {
                            "AppGroupName": "Rick Application Group",
                            "Description": "This is Rick's App group",
                            "FriendlyName": "Rick-App-Group"
                        }
                    ]
                },
                {
                    "HostPoolName": "myHostPool1",
                    "ValidationEnv": false,
                    "AssignmentType": "",
                    "Description": "Mortys Pool",
                    "FriendlyName": "WVD-Host-Pool-1",
                    "Persistent": false,
                    "wvdAppGroup": [
                        {
                            "AppGroupName": "Desktop Application Group",
                            "Description": "This is the default desktop App group",
                            "FriendlyName": "Desktop-App-Group"
                        },
                        {
                            "AppGroupName": "Morty Application Group",
                            "Description": "This is Morty's App group",
                            "FriendlyName": "Morty-App-Group"
                        }
                    ]
                },
                {
                    "HostPoolName": "myHostPool2",
                    "ValidationEnv": false,
                    "AssignmentType": "",
                    "Description": "Jerrys Pool",
                    "FriendlyName": "WVD-Host-Pool-2",
                    "Persistent": false,
                    "wvdAppGroup": [
                        {
                            "AppGroupName": "Desktop Application Group",
                            "Description": "This is the default desktop App group",
                            "FriendlyName": "Desktop-App-Group"
                        },
                        {
                            "AppGroupName": "Jerry Application Group",
                            "Description": "This is Jerry's App group",
                            "FriendlyName": "Jerry-App-Group"
                        }
                    ]
                },
                {
                    "HostPoolName": "myHostPool3",
                    "ValidationEnv": false,
                    "AssignmentType": "",
                    "Description": "Summers Pool",
                    "FriendlyName": "WVD-Host-Pool-3",
                    "Persistent": false,
                    "wvdAppGroup": [
                        {
                            "AppGroupName": "Desktop Application Group",
                            "Description": "This is the default desktop App group",
                            "FriendlyName": "Desktop-App-Group"
                        },
                        {
                            "AppGroupName": "Summer Application Group",
                            "Description": "This is Summer's App group",
                            "FriendlyName": "Summer-App-Group"
                        }
                    ]
                }
            ]
        }
    ]
}
```

---

### Pipeline Definition
Next up, you can find the matching orchestration file `'wvd.custom.pipeline.yml'`. It references all files shown in the [files](#Files) section below. 

First we have the variable templates providing both information for the deployment (like target resource groups, the service connection, etc.).

Next, we have the SBX stage with all it's contained jobs. All jobs will run in parallel as long as the `'dependsOn'` conditions are met. Each job provides all information required by the underlying `'pipeline.jobs.orchestration.deploy.yml'` and could cover pre-deployment, deployment as well as post-deployment steps. 

```YAML
name: WVD deployment

variables:
- template: '../../_Shared-Variables/variables.shared.yaml'
- template: './parameters/variables.shared.modules.yml'

trigger: none

stages:
- stage: SBX
  jobs:

 ##########################
 # BASELINE DEPLOYMENT(s) #
 ##########################
  # Key Vault
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn: ""
      environment: SBX
      moduleName: KeyVault
      moduleVersion: '2019-11-28'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/keyvault.parameters.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      postDeploymentPipelinePath: ./pipeline.steps.keyVault.postDeployment.yml
      resourcegroup: '$(base-rg)'
      isEnabled: true

  # Storage Account + 'File Share Access Group' 1 configuration
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      environment: SBX
      moduleName: StorageAccounts
      moduleVersion: '2019-04-01'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/storageaccount.parameters.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      postDeploymentPipelinePath: ./pipeline.steps.storageAccount.postDeployment.yml
      resourcegroup: '$(base-rg)'
      isEnabled: true
  
  # 'File Share Access Group' 2 configuration
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_StorageAccounts
      environment: SBX
      suffix: 'One'
      moduleName: StorageAccounts
      postDeploymentPipelinePath: ./pipeline.steps.storageAccount.postDeployment1.yml
      isEnabled: true

  ###########################
  # HOST POOL DEPLOYMENT(s) #
  ###########################
  # 'Host Pool' 1 Deployment
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_KeyVault
      - DeployModule_StorageAccounts
      environment: SBX
      moduleName: WVD
      moduleVersion: '2019-12-25'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/wvd.parameters.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      resourcegroup: '$(pool-rg)'
      isEnabled: true

  # 'Host Pool' 2 Deployment
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_KeyVault
      - DeployModule_StorageAccounts
      environment: SBX
      suffix: 'One'
      moduleName: WVD
      moduleVersion: '2019-12-25'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/wvd.parameters1.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      resourcegroup: '$(pool-rg1)'
      isEnabled: true

  # 'Host Pool' 3 Deployment
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_KeyVault
      - DeployModule_StorageAccountsOne
      environment: SBX
      suffix: 'Two'
      moduleName: WVD
      moduleVersion: '2019-12-25'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/wvd.parameters2.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      resourcegroup: '$(pool-rg2)'
      isEnabled: true

  # 'Host Pool' 4 Deployment
  - template: ./resources/pipeline.jobs.orchestration.deploy.yml
    parameters:
      dependsOn:
      - DeployModule_KeyVault
      - DeployModule_StorageAccountsOne
      environment: SBX
      suffix: 'Three'
      moduleName: WVD
      moduleVersion: '2019-12-25'
      parameterFilePath: 'Implementation/Environments/pipeline-orchestrated/wvdShared/parameters/wvd.parameters3.json'
      deploymentPipelinePath:  ./pipeline.steps.deployment.yml
      resourcegroup: '$(pool-rg3)'
      isEnabled: true
```


# Files
Following you can find an overview of the differnt files required for this deployment.
|  |  |  | Description  |
|--|--|--|--|
| wvdShared |  |  |  |
|  | parameters  |  |  |
|  |  | `keyvault.parameters.json` | ARM parameters for the **key vault service** deployment |
|  |  | `storageaccount.parameters.json` | ARM parameters for the **storage account service** deployment  |
|  |  | `variables.shared.modules.yml` | Post-Deployment variables including the `[File Share Access Group]` names |
|  |  | `wvd.parameters.json` | ARM parameters for the 1st **Host Pool** service deployment  |
|  |  | `wvd.parameters1.json` | ARM parameters for the 2nd **Host Pool** service deployment  |
|  |  | `wvd.parameters2.json` | ARM parameters for the 3rd **Host Pool** service deployment  |
|  |  | `wvd.parameters3.json` | ARM parameters for the 4th **Host Pool** service deployment   |
|  | resources |  |  |
|  |  | `pipeline.jobs.orchestration.deploy.yml`  | The general orchestration pipeline job template |
|  |  | `pipeline.steps.deployment.yml`  | The general deployment pipeline step template |
|  |  | `pipeline.steps.keyVault.postDeployment.yml`  | The **key vault** post-deployment step template |
|  |  | `pipeline.steps.storageAccount.postDeployment.yml`  | The 1st **storage account** post-deployment step template. References 1st `[File Share Access Group]` |
|  |  | `pipeline.steps.storageAccount.postDeployment1.yml` | The 2nd **storage account** post-deployment step template. References 2nd `[File Share Access Group]` |
|  | scripts |  |  |
|  |  | `Invoke-GeneralDeployment.ps1` | The logic for the general deployment pipeline step template |
|  |  | `Invoke-KeyVaultPostDeployment.ps1` | The logic for the **key vault** post-deployment pipeline step template |
|  |  | `Invoke-StorageAccountPostDeployment.ps1` | The logic for both **storage account** post-deployment pipeline step templates |
|  | `wvd.custom.pipeline.yml` | | The general pipeline orchestration file |

In addtion, the `wvd.variables.shared.yml` file contains the names of all 5 resource groups. E.g.
```YAML
- name: pool-rg
  value: WVD-Pool-RG

- name: pool-rg1
  value: WVD-Pool-RG1

- name: pool-rg2
  value: WVD-Pool-RG2

- name: pool-rg3
  value: WVD-Pool-RG3

- name: base-rg
  value: WVD-Base-RG
```
