> ![AD.png](/.attachments/AD-74142568-0b3c-4aa2-8c3e-c2a41f5a9629.png =15x)

[[_TOC_]]

---

![deploymentad.png](/.attachments/deploymentad-86cb0c8e-7128-4ddd-9108-5ea76dcc0c4b.png)

The _native AD_ approach requires an additional step in between the _Baseline Deployment_ and the _WVD Host Pool Deployment_. Pausing the process prior to the _WVD Host Pool Deployment_ is necessary in order to store the '_Domain Join User_' credentials in the newly created Key Vault as a secret and to enable Azure Files AD DS Authentication for the Storage Account.

This page provides an overview of all the tasks performed by the flow in the below '_4 - Resource Deployment Overview_' section. The operative steps needed to deploy the solution are then presented in the '_[Deployment operational steps](#deployment-operational-steps)_' section. 

# 4 - Resource Deployment Overview

This section provides an overview of the tasks performed by the flow.

## 4.1 - Baseline Deployment Elements
> ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =20x)

This part of the process is executed by DevOps pipeline using the Service Principal `[Deployment Service Principal]` that was set up in the DevOps Enablement phase. The pipeline contains two deployments that run in parallel.

- Key Vault
  > ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =15x)

  | Name                       | Sub-Step(s)                                                       | Description |
  |---------------------------|--------------------------------------------------------------------|-------------|
  | Key Vault Deployment      |                                                                    | The resource deployment |
  | Key Vault Post-Deployment |                                                                    | |
  |                           | Set secrets for `[Tenant Admin Service Principal]`                 | With the deployment service principal assigned as an owner of the `[Tenant Admin Service Principal]` create a secret for this service principal and store it in the deployed key vault. |
  |                           | Group: Key Vault Access Group                                      | Create a group to enable later reference of secrets in this key vault by the `[Deployment Service Principal]` and/or other users. |
  |                           | Add `[Deployment Service Principal]` to `[Key Vault access group]` | Add the `[Deployment Service Principal]` to this group to enable read access. |
  |                           | Set access policies for `[Key Vault access group]`                 | Add the created `[Key Vault access group]` to the deployed key vault's access policies to enable read access on the key vault. |

- Storage Account
  > ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x)

  | Name                            | Sub-Step(s)                                                  | Description |
  |---------------------------------|--------------------------------------------------------------|-------------|
  | Storage Account Deployment      |                                                              | The resource deployment creating a storage account with two blob containers, <br> - one for post-deployment scripts (access by the WVD VMs' custom script extension)<br> - and one for software components to be installed on the WVD VMs,<br> and one file share, hosting WVD user profiles. |
  | Storage Account Post-Deployment |                                                              | |
  |                                 | Set SMB Contributor Role (IAM) for `[File Share Access Group]` | Assigns the '_Storage File Data SMB Share Contributor_' RBAC role to the `[File Share Access Group]` group to enable the profile synchronization  from the service side. The `[File Share Access Group]` is an AD group in sync with Azure AD. |
  |                                 | Set Blob Content                                             | Upload all required post deployments scripts and the `'FSLogix.exe'` installer to the corresponding containers in the storage account. |

>**Note**: By default this solution installs and executes FSLogix on the WVD VMs. If you prefer to opt out from FSLogix adoption, you have to disable the task responsible for deploying the storage account in the pipeline, as detailed in the _Deployment operative steps_ at the end of this page.
## 4.2 - KV and Storage Account Config Elements

### WVDADAuthPreReq PowerShell Module
> ![powershell.png](/.attachments/powershell-8596a337-15e8-4bcb-b0d6-cb3d02d382d0.png =20x)

> **NOTE**: The module has the default command prefix _'WVDADAuthPreReq'_, i.e. once imported, you must invoke all functions of the public folder (exported functions) with the prefix. E.g. `'Get-Name'` becomes <code>'Get-<b>WVDADAuthPreReq</b>Name'</code>

Once imported, the _WVDADAuthPreReq_ module provides the following functions:

- Set-DomainJoinSecret
- Set-AzStorageADAuth

#### Set-DomainJoinSecret 

As detailed in the [Prerequisites](/Delivery-Guide/2019-GA-Release/Deployment/Pre%2Ddeployment/Prerequisites) section, this function must be run by the _**WVD KV Secret Admin**_. Refer to that section to verify all permissions are in place before proceeding.

This adds the current logged-in user '_WVD KV Secret Admin_' to the `[Key Vault Access Group]` previously deployed by the pipeline.
Next, it stores the given '_Domain join user_' password as a secret in the given key vault.

- **Parameters**

   | Parameter | Description |
   | --------- | ----------- |
   | SubscriptionId | The id of the Azure subscription |
   | DomainJoinUserName | Name of the domain join user, e.g. WVDJoinDomain |
   | DomainJoinPrincipalName | Name of the domain join user principal name, e.g. WVDJoinDomain@contoso.com |
   | DomainJoinPassword | Password of the domain join user |
   | KeyVaultName | The key vault to store the domain join user password in |
   | KeyVaultAccessGroup | The group `[Key Vault Access Group]` having access to the key vault |

  
- Example

   ```PowerShell
   # Store the 'Domain Join User' credentials in the key vault

   # Prepare input 
   $subscriptionId          = "[subscriptionId]"
   $domainJoinUserName      = "[domainJoinUserName]"
   $domainJoinPrincipalName = "[domainJoinPrincipalName]"
   $domainJoinPassword      = "[domainJoinPassword]"
   $keyVaultName            = "[keyVaultName]"
   $keyVaultAccessGroup     = "[keyVaultAccessGroup]"
	
   $inputObject = @{
      SubscriptionId          = $subscriptionId 
      DomainJoinUserName      = $domainJoinUserName 
      DomainJoinPrincipalName = $domainJoinPrincipalName 
      #This converts your 'Domain Join User' password string to a SecureString
      DomainJoinPassword      = (ConvertTo-SecureString -String $domainJoinPassword -AsPlainText -Force)
      KeyVaultName            = $keyVaultName 
      KeyVaultAccessgroup     = $keyVaultAccessGroup
   }

   #Execute
   Set-WVDADAuthPrereqDomainJoinSecret @inputObject
   ```
 

#### Set-AzStorageADAuth

As detailed in the [Prerequisites](/Delivery-Guide/2019-GA-Release/Deployment/Pre%2Ddeployment/Prerequisites) section, this function must be run on a _**Domain Joined Machine**_ by the _**WVD AD Auth Admin**_. Refer to that section to verify all permissions are in place before proceeding.

This enables Active Directory authentication over SMB for Azure file shares. The setup includes the import of the AzFilesHybrid module, the creation of an identity representing the storage account in your AD, and finally enabling the AD auth for Azure Files feature on your storage account

- Parameters

   | Parameter | Description |
   | --------- | ----------- |
   | SubscriptionId | The id of the subscription |
   | StorageAccountRGName | The name of the resource group containing the target storage account |
   | StorageAccountName | The name of the target storage account |
   | DomainAccountType | The type of AD object to be used either a service logon account (user account) or a computer account,depending on the AD permission you have and preference. Choose between `'ComputerAccount'` and `'ServiceLogonAccount'` |
   | OrganizationalUnitName | The organizational unit for the AD object to be added to. This parameter is optional, but many environments will require it. |
   | PathToAzFilesHybridManifest | (opt) Path to the manifest of the AzFilesHybrid module to import in order to enable the feature. The default is the one stored into this module's static folder. |
   | PathToAzFilesHybridModule | (opt) Path to the AzFilesHybrid module to import in order to enable the feature The default is the one stored into this module's static folder. |


- Example

   ```PowerShell
   #Enable Active Directory authentication over SMB for Azure file shares for the target storage account

   # Prepare input 
   $subscriptionId        = "[subscriptionId]"
   $storageAccountRGName  = "[storageAccountRGName]"
   $storageAccountName    = "[storageAccountName]"
   $domainAccountType     = "[ComputerAccount|ServiceLogonAccount]"
   $orgUnit               = "[orgUnit]"
	
   $inputObjectADAuth = @{  
      SubscriptionId         = $subscriptionId 
      StorageAccountRGName   = $storageAccountRGName
      StorageAccountName     = $storageAccountName
      DomainAccountType      = $domainAccountType
      OrganizationalUnitName = $orgUnit
   }

   #Execute
   Set-WVDADAuthPrereqAzStorageADAuth @inputObjectADAuth
   ```

## 4.3 - Host Pool Deployment

> ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =20x)

- WVD
  > ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =15x)

  The host pool deployment itself consists solely of the ARM deployment of the WVD host pool. That is, a set of virtual machines registered as session hosts in the WVD tenant.
The post-deployment step in the diagram acts as a placeholder for further important actions. In this case for example the assignment of all users of the `[File Share Access Group]` to AppGroups. This and other actions are further elaborated in the ['Post-Deployment' section](https://dev.azure.com/SecInfra/Components/_wiki/wikis/Start%20Here/689/Post-deployment). 

# Deployment operational steps

This section lists the deployment operative steps needed to deploy the solution. 

Prior to running the deployment, double check that you have completed all the required prerequisites presented in the previous sections. 

Make sure you correctly setup all parameter files based on the target environment you plan to deploy, as detailed [here](/Delivery-Guide/2019-GA-Release/Deployment/Resource-Deployment/Preparation). In particolar, pay attention to the ones enabling **FSLogix** installation and configuration on the target WVD VMs and to the ones determining the **user persistency** of your machines, i.e. personal vs pooled desktops. 

Follow the steps below to deploy the WVD automation for the native AD approach.
1. **Baseline Deployment**
   > ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =20x) ![arrow.svg](/.attachments/arrow-96bc8d6f-6f36-46b5-b500-80f578180776.svg =45x) ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =20x) ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =20x)

   1.1. Navigate to _DevOps Pipelines_, locate and access the pipeline you set up as described [here](/Delivery-Guide/2019-GA-Release/Deployment/Pre%2Ddeployment/DevOps-Enablement).

   1.2.  Update the `'Environments/pipeline-orchestrated/wvdShared/custom.pipeline.yml'` file with one of the following set of values for the `'isEnabled'` flag in the three jobs. 
Depending on if you decide to adopt FSLogix or not, you may decide to disable the deployment of the storage account, whose main goal is to store FSLogix related resources such as the WVD profiles share.

   _**With FSLogix**_

   | Job | Flag | Value |
   |-----|------|-------|
   | KeyVault | `'isEnabled'` | `true` |
   | StorageAccounts| `'isEnabled'` | `true` |
   | WVD | `'isEnabled'` <br/>  | `false` |

   > That will deploy the Key Vault and the Storage Account in your target subscription.

   _**Without FSLogix**_

   | Job | Flag | Value |
   |-----|------|-------|
   | KeyVault | `'isEnabled'` | `true` |
   | StorageAccounts| `'isEnabled'` | `false` |
   | WVD | `'isEnabled'` <br/>  | `false` |

   > That will deploy the Key Vault in your target subscription. 

   The deployment of the WVD host pool is skipped at this stage, as additional configurations are needed on the key vault and storage account prior to the WVD deployment.

   1.3 Execute the pipeline and wait for the deployment completion

   ![image.png](/.attachments/image-1397fd25-67a3-46d8-845c-e8ef57fbb051.png =85%x)

2. **KV and Storage Account config**
   > ![powershell.png](/.attachments/powershell-8596a337-15e8-4bcb-b0d6-cb3d02d382d0.png =20x) ![arrow.svg](/.attachments/arrow-96bc8d6f-6f36-46b5-b500-80f578180776.svg =45x) ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =20x) ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =20x)

   2.1. Logon on a _**Domain-joined machine**_ 

   2.2. Open a PowerShell session and verify that the required PowerShell modules, listed above, have been installed.

   2.3. Import the WVDADAuthPrereq Module

   ```PowerShell
    Import-Module \path\to\WVDADAuthPrereq.psd1
   ```

   2.4. Configure key vault: execute the `Set-DomainJoinSecret`

   ```PowerShell
    Set-WVDADAuthPrereqDomainJoinSecret
   ```

   When prompted for sign-in to Azure, use the **_WVD KV Secret Admin_** user.

   2.5. Configure storage account: execute the `Set-AzStorageADAuth` to enable Azure Files AD DS Authentication preview on your storage account. Make sure your storage account is in a [supported region](https://docs.microsoft.com/it-it/azure/storage/files/storage-files-identity-auth-active-directory-enable#regional-availability).

    ```PowerShell
    Set-WVDADAuthPrereqAzStorageADAuth 
   ```

   When prompted for sign-in to Azure, use the **_WVD AD Auth Admin_** user.

>**Warning**: Update AD account password
If you registered the AD identity/account representing your storage account under an OU that enforces password expiration time, you must rotate the password before the maximum password age. Failing to update the password of the AD account will result in authentication failures to access Azure file shares. In that case refer [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable#update-ad-account-password) for additional details.

3. **Host Pool Deployment Elements**
   > ![svg-azure-pipelines.svg](/.attachments/svg-azure-pipelines-59700d42-62ab-40af-bd21-e15c5cb490cd.svg =20x) ![arrow.svg](/.attachments/arrow-96bc8d6f-6f36-46b5-b500-80f578180776.svg =45x) ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =20x)

   3.1 Navigate to _DevOps Pipelines_, locate and access the pipeline you set up as described [here](/Welcome/Components/WVD-Automation/Deployment-%2D-Overview/Pre%2Ddeployment/DevOps-Enablement).

   3.2 Update the `'Environments/pipeline-orchestrated/wvdShared/custom.pipeline.yml'` file with the following values for the `'isEnabled'` flag in the three jobs:

   | Job | Flag | Value |
   |-----|------|-------|
   | KeyVault | `'isEnabled'` | `false` |
   | StorageAccounts| `'isEnabled'` | `false` |
   | WVD | `'isEnabled'` <br/>  | `true` |

   > That will deploy the WVD host pool VMs in your target subscription. 
  
   3.3 Execute the pipeline and wait for the deployment completion

   ![image.png](/.attachments/image-0946fdc9-2c3f-47ea-adc7-754544cd4b22.png =85%x)

After the desired resources have been deployed, follow guidelines in the [Post-Deployment](/Welcome/Components/WVD-Automation/Deployment-%2D-Overview/Post%2Ddeployment) section to manage user and _RemoteApps_ assignment.

---

> **Note**: Please refer to the official docomentation [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable#assign-access-permissions-to-an-identity) for further details.