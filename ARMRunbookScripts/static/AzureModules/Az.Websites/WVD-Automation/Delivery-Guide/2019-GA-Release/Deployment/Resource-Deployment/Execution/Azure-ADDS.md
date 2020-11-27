
> ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-9a3d7522-7ba9-47d8-aca8-b2ff9a03c274.svg =15x)

With all infrastructure components, parameter and variable files in place, the AADDS deployment does not require any further configuration.

[[_TOC_]]

![deploymentaadds.png](/.attachments/deploymentaadds-cc72186f-a5bd-4e47-b680-b81e4f790d21.png)

# Elements
Before we trigger the deployment, let us take a look on the individual performed actions:
## 4.1 - Baseline Deployment Elements
This part of the process contains two parallel deployments:
- Key Vault
  > ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =15x)

  | Name                       | Sub-Step(s)                                                       | Description |
  |---------------------------|--------------------------------------------------------------------|-------------|
  | Key Vault Deployment      |                                                                    | The service deployment |
  | Key Vault Post-Deployment |                                                                    | |
  |                           | User: Domain Join User                                             | Create an AzureAD user to perform later domain joins for us |
  |                           | Set secrets for `[Domain Join User]`                               | A password for this user is created and stored in the deployed key vault for later ARM reference |
  |                           | Set secrets for `[Tenant Admin Service Principal]`                 | With the deployment service principal assigned as an owner of the `[Tenant Admin Service Principal]` create a secret for this service principal an store it in the deployed key vault |
  |                           | Group: Key Vault Access Group                                      | Create a group to enable later reference of secrets in this key vault by the deployment service principal and or other users |
  |                           | Add `[Deployment Service Principal]` to `[Key Vault access group]` | Add the deployment service principal to this group to enable read access |
  |                           | Set access policies for `[Key Vault access group]`                 | Add the created `[Key Vault access group]` to the deployed key vault's access policies to enable read access on the key vault |

- Storage Account Deployment
  > ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x)

  | Name                            | Sub-Step(s)                                                  | Description |
  |---------------------------------|--------------------------------------------------------------|-------------|
  | Storage Account Deployment      |                                                              | The service deployment with two containers (one for scripts, one for software) and one file share (for user context folders) |
  | Storage Account Post-Deployment |                                                              | |
  |                                 | Group: File Share Access Group                               | Creates a group to host all users who's user profiles should be synced with tis storage account's file share |
  |                                 | Set SMB Contributor Role (IAM) for `[File Share Access Group]` | Assigns the 'SMB Contributor' RBAC role to the created group to enable the profile synchonization  from the service side |
  |                                 | Set Blob Content                                             | Upload all required post deployments scripts and the `'FSLogix.exe'` to the corresponding containers in the storage account. |

## 4.3 - Host Pool Deployment Elements
> ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =15x)

The host pool deployment itself consists solely of the ARM deployment of the WVD host pool. That is, a set of virtual machines registered as session hosts in the WVD tenant.
The post-deployment step in the diagram acts as a placeholder for further important actions. In this case for example the assignment of all users of the `[File Share Access Group]` to AppGroups. This and other actions are further elaborated in the ['Post-Deployment' section](https://dev.azure.com/SecInfra/Components/_wiki/wikis/Start%20Here/689/Post-deployment). 


# Deployment operational steps

This section lists the deployment operative steps needed to deploy the solution. 

Prior to running the deployment, double check that you have completed all the required prerequisites presented in the previous sections. 

Make sure you correctly setup all parameter files based on the target environment you plan to deploy, as detailed [here](/Welcome/Components/WVD-Automation/Deployment-%2D-Overview/Resource-Deployment/Preparation). In particolar, pay attention to the ones enabling **FSLogix** installation and configuration on the target WVD VMs and to the ones determining the **user persistency** of your machines, i.e. personal vs pooled desktops. 

Follow the steps below to deploy the WVD automation for the AADDS approach.
1. **Baseline Deployment**
   > ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =20x) ![arrow.svg](/.attachments/arrow-96bc8d6f-6f36-46b5-b500-80f578180776.svg =45x) ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =20x) ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =20x) ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =20x)

   1.1. Navigate to _DevOps Pipelines_, locate and access the pipeline you set up as described [here](/Welcome/Components/WVD-Automation/Deployment-%2D-Overview/Pre%2Ddeployment/DevOps-Enablement).

   1.2.  Update the `'Environments/pipeline-orchestrated/wvdShared/custom.pipeline.yml'` file with one of the following set of values for the `'isEnabled'` flag in the three jobs. 
Depending on if you decide to adopt FSLogix or not, you may decide to disable the deployment of the storage account, whose main goal is to store FSLogix related resources such as the WVD profiles share.

   _**With FSLogix**_

   | Job | Flag | Value |
   |-----|------|-------|
   | KeyVault | `'isEnabled'` | `true` |
   | StorageAccounts| `'isEnabled'` | `true` |
   | WVD | `'isEnabled'` <br/>  | `true` |

   > That will deploy the Key Vault, the Storage Account and the WVD host pool VMs in your target subscription.

   _**Without FSLogix**_

   | Job | Flag | Value |
   |-----|------|-------|
   | KeyVault | `'isEnabled'` | `true` |
   | StorageAccounts| `'isEnabled'` | `false` |
   | WVD | `'isEnabled'` <br/>  | `true` |

   > That will deploy the Key Vault and the WVD host pool VMs in your target subscription.

   1.3 Trigger the pipeline and wait for the deployment completion

Once all resources are deployed, you can set the `'isEnabled'` flag for both the key vault and the storage account tasks to `'false'`. Also, out-comment the depends flag of the WVD job. This makes it faster to deploy new Host-Pools as the pre-conditions are already set up successfully.