This section describes the individual components deployed into the _Profiles Resource Group_, when they are needed and what parameters are required for each.

[[_TOC_]]

<span style="color:red">[<TODO>: ADD Profiles IMAGE]</span>

---

<div style="padding: 10px; border-left: 1px solid #ED7D31;" >

![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) **NOTE:** Only required in case the FSLogix user data synchronization is deployed

# Deployments

## Profiles Storage Account
> ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =25x)

This storage account Storage hosts file shares used to synchronize user profiles across machines using FSLogix. As such, it needs the NativeAD/AADDS authentication enabled.

### Related Deployments
|  | **Type** | **Name**  | **Description** |
|--|----------|-----------|-----------------|
| ![Icon-storage-400-Azure-Fileshare.png](/.attachments/Icon-storage-400-Azure-Fileshare-c17002f0-a33f-46e8-b014-185e3ef9c6e5.png =15x) | Storage Account File Share | File Share |  Hosts the user data synced by FSLogix. |
| ![rbac.png](/.attachments/rbac-c74dc4b3-d9aa-4ed2-a9a9-abf9c7812751.png =15x) | Role Assignment | SMB Contributor [WVD User Group] | One of the requirements to be met to enable the user data sychronization. |

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `storageAccountName` | string | - | <Globally unique value> | Required | The name of the profile storage account (e.g. "profilestorageaccount"). | 
| `fileShares` | array | - | - | Required | Should contain the definition for the file shares to create to be used by FSLogix as well as the RBAC rules that are required alongside them. |

### Required Variables
_None_

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **[MANUAL TASK] Key Vault and Storage Account config**

   <div style="padding: 10px; border-left: 1px solid #FFC000;" >

   ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) **NOTE**: Only required for Native-AD authentication

   > ![powershell.png](/.attachments/powershell-8596a337-15e8-4bcb-b0d6-cb3d02d382d0.png =20x) ![arrow.svg](/.attachments/arrow-96bc8d6f-6f36-46b5-b500-80f578180776.svg =45x) ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =20x) ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =20x)
   
   1.  Logon on a _**Domain-joined machine**_ 

   2. Open a PowerShell session and verify that the required PowerShell modules, listed above, have been installed.

   3. Import the WVDADAuthPrereq Module

      ```PowerShell
      Import-Module '.\WVDADAuthPrereq.psd1'
      ```

    4. Configure key vault: execute the `Set-DomainJoinSecret` <span style="color:red">[<TODO>: @<603DE783-D07C-6BF7-8416-0B68F35E94EB>, is this still required? / Refactor?]</span> 
       ```PowerShell   
        Set-WVDADAuthPrereqDomainJoinSecret
        ```
        When prompted for sign-in to Azure, use the **_WVD KV Secret Admin_** user.

     5. Configure storage account: execute the `Set-AzStorageADAuth` to enable Azure Files AD DS Authentication preview on your storage account. Make sure your storage account is in a [supported region](https://docs.microsoft.com/it-it/azure/storage/files/storage-files-identity-auth-active-directory-enable#regional-availability).

        ```PowerShell
        Set-WVDADAuthPrereqAzStorageADAuth 
        ```

        When prompted for sign-in to Azure, use the **_WVD AD Auth Admin_** user.

      >**Warning**: Update AD account password
If you registered the AD identity/account representing your storage account under an OU that enforces password expiration time, you must rotate the password before the maximum password age. Failing to update the password of the AD account will result in authentication failures to access Azure file shares. In that case refer [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable#update-ad-account-password) for additional details.
</div>
</div>