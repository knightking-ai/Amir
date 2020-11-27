This WVD automation solution covers two main scenarios (Native AD or Azure ADDS authentication), based on the customer's identity and access management solution. Depending on their choice, different prerequisites have to be met. 

The following three paragraphs describe
- **Shared prerequisites** which are common to the two approaches
- Distinct prerequisites:
  - **Native AD prerequisites** (in case 'traditional' ADDS is used) covering resources that must be in place before the WVD automation is deployed - **OR** -
  - **Azure ADDS prerequisites** which are needed to integrate in case Azure ADDS is adopted.

> **NOTE**: 
> **ALL steps explained in this section have to be performed by the customer**, before the WVD offer's delivery starts. We only validate that these prerequisites are in place.

[[_TOC_]]

![prereqpre.png](/.attachments/prereqpre-03b9bab2-06e5-4253-ace9-58ffacb6fa2f.png)

---

# Shared Prerequisites
The following prerequisites are the ones outlined in green in the above diagram and are common to both approaches.
- [Users](#Users)
- [Azure Environment](#Azure-Environment)

## Users

> ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x)
- **Global Administrator**
All the steps within the WVD Enablement phase must be performed on behalf of a user who holds the _Global Administrator_ role in AAD.

## Azure Environment

> ![azure-logo.png](/.attachments/azure-logo-df90809a-7022-452d-b5e5-4f0d777dc45b.png =15x)
- ![Icon-networking-61-Virtual-Networks.svg](/.attachments/Icon-networking-61-Virtual-Networks-527ab1d3-5066-45fb-a793-0bc3430dcb16.svg =15x) _**VirtualNetwork**_
One ore more Virtual Networks with subnet(s) of sufficient size for one or multiple host pool deployments have to be prepared by the customer. Assumption: routing and firewall configuration (i.e. NSGs and/or NVAs) are in place and configured according to the customer's standards. 

  >**NOTE**: the WVD VMs placed in the VNET(s) provided have to be able to communicate to the WVD service and key Azure platform services on TCP 443 (outbound), as well as to the Azure KMS service on TCP 1688 (outbound).
- ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) _**Windows Virtual Desktop Applications**_
As both the '_Windows Virtual Desktop_' and '_Windows Virtual Desktop Client_' AAD applications must be registered in the AAD tenant prior to any deployment, the _Global Administrator_ must register both using the corresponding [website](https://rdweb.wvd.microsoft.com/). See the details in the [WVD Service Enablement](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation/711/WVD-Service-Enablement) chapter.

---

# Distinct prerequisites

The customer has to provide one of two identity solutions:
- **Either** we require a configured ['Native' **_ADDS_**](#Native-AD-prerequisites) (AD, Active Directory) environment to enable an authentication with our file share.](#Native-AD-prerequisites)  This can be achieved either via deploying Domain Controllers to the cloud or 'just' extending the on-premises domain by allowing network level access to the Domain Controllers from the WVD subnets (via VPN or ExpressRoute)
- **or** an [_**Azure ADDS**_](#Azure-ADDS-Prerequisites) domain.

## Native AD prerequisites
> ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x)

The following prerequisites are the ones outlined in yellow in the above diagram and having a small DC icon on the top right corner .


### Identity Solution

- ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) ![Icon-identity-221-Azure-Active-Directory.svg](/.attachments/Icon-identity-221-Azure-Active-Directory-647d4514-695c-43e0-8137-c336445e4388.svg =15x) _**AD sync to Azure AD**_

   A Windows Server Active Directory in sync with Azure Active Directory. This can be configured with Azure AD Connect.

### Users and Groups
All **AD identities** used to access Azure file shares must be **synced to Azure AD** to enforce share level file permissions through the standard role-based access control (RBAC) model. In particular:

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**Domain Join User**_

  This is the Native AD user that can join machines to the domain and will perform the domain join of the WVD host pool's VMs as part of the WVD host pool deployment. The password of this user will be stored as a key vault secret by the _**WVD KV Secret Admin**_.

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**WVD KV Secret Admin**_

  This user will perform the steps detailed in the [Native AD execution](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation/714/Native-AD) section and store the Native AD _**Domain Join User**_ password in a key vault secret. 
  - Required Permissions (Azure)

    | Scope| Configuration | Role |
    |--|--|--|
    | Subscription | RBAC (IAM) |  `Key Vault Contributor` |
    | AzureAD | Roles | `Groups Administrator` |

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**WVD AD Auth Admin**_

  This user will perform the steps detailed in the [Native AD execution](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation/714/Native-AD) section on a _**Domain Joined Machine**_ and enable the recently introduced (public preview) support of leveraging Active Directory Domain Service for authentication to Azure file shares.

  - Required Permissions (Azure)

    | Scope| Configuration | Role |
    |--|--|--|
    | Storage Account Resource Group| RBAC (IAM) | `Contributor` |
    
  - Required Permissions (AD)
Create a service logon account or a computer account in the target AD.

  >**Note**: This prerequisite is needed for enabling Azure Files AD DS Authentication (public preview) and leverage it for profile management using **FSLogix**.

- ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) _**File Share Access Group**_

  This is the AD group synced to Azure AD and containing users who will be assigned to the WVD host pool (i.e. WVD users). 
As detailed in the following sections, this group will be assigned with proper share-level permissions with RBAC and with NTFS permissions as part of the Storage Account deployment.

  >**Note**: This prerequisite is needed for enabling Azure Files AD DS Authentication (public preview) and leverage it for profile management using **FSLogix**.

### Additional resources 

- ![Icon-compute-21-Virtual-Machine.svg](/.attachments/Icon-compute-21-Virtual-Machine-347d68cb-caf8-41aa-a24e-a13d5dfa3ef6.svg =15x) _**Domain Joined machine**_

  This can be an on-premises machine or an Azure VM and must be domain joined to AD (also referred as AD DS). 
This machine will be used by the _**WVD AD Auth Admin**_ to enable Active Directory Domain Service for authentication to Azure file shares. Connectivity to Azure is required.

  >**Note**: This prerequisite is needed for enabling Azure Files AD DS Authentication (public preview) and leverage it for profile management using **FSLogix**.

### Supported Regions
For optimal performance, it is recommend to deploy the storage account in the same region as the VM from which you plan to access the share.

At the time of writing, Azure Files AD authentication (preview) is available in most public regions, but is **not** yet available in:

- West US


 > **Note**: For further details or possible updates, please refer [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable#regional-availability).

## Azure ADDS prerequisites
> ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x)

The following prerequisites are the ones outlined in light blue in the above diagram and having a small Azure ADDS icon on the top right corner ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x).

> **NOTE**: this scenario also utilizes a couple of services users, but these are programmatically configured during the deployment phase.

### Identity Solution
  - ![azure-logo.png](/.attachments/azure-logo-df90809a-7022-452d-b5e5-4f0d777dc45b.png =15x) ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x) _**Azure ADDS**_ 

    If the customer wants to leverage _Azure ADDS_ for the solution, it must be fully set up and configured prior to the WVD project delivery begins.