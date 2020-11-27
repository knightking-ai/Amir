This section describes the individual components deployed into the _Management Resource Group_, when they are needed and what parameters are required for each.

[[_TOC_]]

<span style="color:red">[<TODO>: ADD Mgmt IMAGE]</span>

---

# Deployments

## Assets Storage Account
> ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =25x)

This storage account has several containers with the files necessary for several deployments like the Host-Pool deployment CSE, or Automation Account Runbook scaling script.

### Related Deployments
|  | **Type** | **Name**  | **Description** |
|--|----------|-----------|-----------------|
|![azure blob storage.png](/.attachments/azure%20blob%20storage-4f13c72a-b38d-4cd6-a2b5-019937c540fb.png =15x) | Blob container | _wvdscripts_ | Hosts scripts to be downloaded to the session host when deployments |
|![azure blob storage.png](/.attachments/azure%20blob%20storage-4f13c72a-b38d-4cd6-a2b5-019937c540fb.png =15x) | Blob container | _wvdsoftware_ | Hosts software to be downloaded to the session hosts when deployed |
|![azure blob storage.png](/.attachments/azure%20blob%20storage-4f13c72a-b38d-4cd6-a2b5-019937c540fb.png =15x) | Blob container | _wvdscaling_ | Hosts scripts to be referenced during the scaling runbook deployment |

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `storageAccountName` | string | - | <Globally unique value> | Required | The name of the storage account (e.g. "assetsStorageAccount"). | 

### Required Variables
| Variable Name | Type | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `storageAccountName` | string | The name of the storage account (e.g. "assetsStorageAccount"). | 

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Set Blob Content**
   - **_Description:_** To make all required files available during the deployment of this solution, they first must be uploaded to this storage account. It is recommended to use the subsequent script as a post-deployment task in the pipeline.
   - **_Script:_** `Set-WVDBlobContent.ps1` 
   
---

## Wvd Key Vault
> ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =25x)

A key vault is created to store several secrets used during the deployments. i.e. the '_Domain Join Service Account'.

### Related Deployments
_None_

### Required Parameters

| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| keyVaultName | string | - | - | Required | The name of the key vault (e.g. "wvd-kvlt"). |
| accessPolicies | arary | - | - | Required | The access policies to add. Should contain at least the deployment service principal. |
| enableVaultForDeployment | bool | `true` | [true/false] | Optional |Specifies whether Azure Virtual Machines are permitted to retrieve certificates stored as secrets from the key vault. |
| enableVaultForTemplateDeployment | bool | `true` | [true/false] | Optional | Specifies whether Azure Resource Manager is permitted to retrieve secrets from the key vault. |

### Required Variables
| Variable Name | Type | Description |
| :- | :- | :- |
| `vaultName` | string | Name of the deployed key vault |
| `domainJoin_userName` | string | Name of the user responsible to perform later domain joins |
| `domainJoin_pwd` | securestring | Password for the given domain join user |

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Set secret for Domain Join User**
   - **_Description:_** As the WVD Session Host VMs must be domain joined the customer must provide us with a DevOps variable as described in the [DevOps Enablement](/Delivery-Guide/Latest-Release/Deployment/Pre%2DDeployment/DevOps-Enablement#storing-secrets-in-devops-_variable-group_). Using the subsequent script in the pipeline, you can automatically fetch this value and upload it to the designated key vault.
   - **_Script:_** `Add-KeyVaultCustomOrGeneratedSecret.ps1`     
</p>

2. **Set secret for Default VM Admin**
   - **_Description:_** As the session host VM requires a local admin password, you can either have one generated or provide it as described in the [DevOps Enablement](/Delivery-Guide/Latest-Release/Deployment/Pre%2DDeployment/DevOps-Enablement#storing-secrets-in-devops-_variable-group_). Using the subsequent script in the pipeline, you can automatically fetch this value and upload it to the designated key vault.
   - **_Script:_** `Add-KeyVaultCustomOrGeneratedSecret.ps1`  

---

## Profiles Recovery Services Vault (RSV)
> ![Recovery Services Vaults.svg](/.attachments/Recovery%20Services%20Vaults-3297985c-7e03-4ed2-8f7e-d2845d5e7a2e.svg =25x) 

<div id="demobox" style="padding: 10px; border-left: 1px solid #ED7D31;">

![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) **NOTE:** Only required in case the FSLogix user data synchronization is deployed

Can be deployed to backup the File-Shares populated by FSLogix.

### Related Deployments
|  | **Type** | **Name**  | **Description** |
|--|----------|-----------|-----------------|
| ![backupPolicy.png](/.attachments/backupPolicy-a5fe3999-95a9-4d98-959e-5223b9fcf2c8.png =15x) | Backup Policy | Profiles Policy | This policy is created to control how the FileShare backup should be done |
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Protection Container Link | - | Before the FileShares can be added as BackupItems to the RSV, a connection to the encapsulating storage account must be established. Only a single RSV can have a connection to a specific storage account at any given time. |

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| recoveryVaultName | string | - | - | Required | The name of the recovery services vault (e.g. "profilesBackupVault"). |
| backupPolicies | array | - | - | Required | Should contain a definition of a backup policy to control the FileShare backup |
| protectionContainers | array| - | - | Required | Should contain the link to the FileShare storage account. |

### Required Variables
| Variable Name | Type | Description |
| :- | :- | :- |
| `recoveryServicesVaultName` | string | Name of the RSV to backup the items in |
| `RecoveryServicesVaultResourceGroup` | string | Resource group name of the RSV to backup the items in  |
| `fileSharePolicyMaps` | Hashtable[] |  An array of items to backup. Each item needs the 'policyName' specified and a list of backup 'items' specified. Each of those backup items needs a 'StorageAccountName' specified an optionally 'FileShareNames' as a comma separated list. |

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Link protected items**
   - **_Description:_** Unfortunately, currently the ARM deployment to link protected items (like the FilesShares) to the RSV does not work. Hence, we are forced to perform this task via PowerShell using the subsequent script. It is recommended to reference it from the pipeline inside a post-deployment step.
   - **_Script:_** `Set-RsvProtectedItemsConfiguration.ps1`

</div>

---

## Scaling Automation Account
>  ![Icon-manage-322-Automation-Accounts.svg](/.attachments/Icon-manage-322-Automation-Accounts-18077721-1b49-42d4-b5bd-9b1a8f0becf7.svg =25x) 

Contains the scaling runbook and is triggered by the 'WVD Scaling Scheduler(s)'. Responsible for scaling any given host pool. 

### Related Deployments
|  | **Type** | **Name**  | **Description** |
|--|----------|-----------|-----------------|
| ![runbook.png](/.attachments/runbook-1e86f61e-e39a-43b9-821c-56070d9152f0.png =15x) | Runbook | ScalingRunbook | The scaling runbook deployment. |

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| automationAccountName | string | - | - | Required | The name of the automation account (e.g. "wvd-scaling-autoaccount"). |
| runbooks | array | - | - | Required | Should contain the definition of the scaling runbook that links to the scaling script found in the assets storage account. |

### Required Variables
| Variable Name | Type | Description |
| :- | :- | :- |
| `AutomationAccountName` | string | Name of the automation account |
| `ScalingRunbookName` | string | Name of the scaling runbook |
| `WebhookName` |string | Name of the runbook webhook to create |
| `RunAsConnectionSPName` | string | Name of the service principal to create that acts as the RunAs service principal for the automation account |
| `KeyVaultName` | string | Name of the key vault to use for required secrets |
| `RunAsSelfSignedCertSecretName` | string | Name of the secret for the cert password |
| `AutoAccountRunAsCertExpiryInMonths` | int | Amount of months the certificate for the run as account should remain valid |
| `tempPath` | string | A path to store files in that are created during execution. |

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Configure Automation Account**
   - **_Description:_** To get the WVD scaling working a little further configuration is required. First, we have to create a webhook for the scaling runbook and store it in the WVD key vault and second upload all modules that are required by the runbook logic.
   - **_Script:_** `Set-AutomationAccountConfiguration`
</p>

2. **Configure 'RunAs' account**
   - **_Description:_** To enable the automation account to work properly we need to set up and configure a 'RunAs' as account (aka _Automation Account Service Principal_). We recommend to invoke the subsequent script in a post-deployment task in the pipeline.
   - **_Script:_** `New-RunAsAccount`
</p>