This WVD DevOps automation solution covers two main scenarios (Native AD or Azure ADDS authentication), based on the customer's identity and access management solution. Depending on their choice, different prerequisites have to be met. 

The following three paragraphs describe
- **Shared prerequisites** which are common to the two approaches
- Distinct prerequisites:
  - **Native AD prerequisites** (in case 'traditional' ADDS is used) covering resources that must be in place before the WVD DevOps automation is deployed - **OR** -
  - **Azure ADDS prerequisites** which are needed to integrate in case Azure ADDS is adopted.

> **NOTE**: 
> **ALL steps explained in this section have to be performed by the customer**, before the WVD offer's delivery starts. We only validate that these prerequisites are in place.

[[_TOC_]]

<span style="color:red">[<TODO>: ADD PRE-REQ IMAGE]</span>

---

# Shared Prerequisites
The following prerequisites are the ones outlined in green in the above diagram and are common to both identity approaches.
- [Azure Environment](#Azure-Environment)

## Azure Environment

> ![azure-logo.png](/.attachments/azure-logo-df90809a-7022-452d-b5e5-4f0d777dc45b.png =15x)

- ![Icon-networking-61-Virtual-Networks.svg](/.attachments/Icon-networking-61-Virtual-Networks-527ab1d3-5066-45fb-a793-0bc3430dcb16.svg =15x) _**VirtualNetwork**_
One ore more Virtual Networks with subnet(s) of sufficient size for one or multiple host pool deployments have to be prepared by the customer. Assumption: routing and firewall configuration (i.e. NSGs and/or NVAs) are in place and configured according to the customer's standards. 

  >**NOTE**: the WVD VMs placed in the VNET(s) provided have to be able to communicate to the WVD service and key Azure platform services on TCP 443 (outbound), as well as to the Azure KMS service on TCP 1688 (outbound).

---
---

# Distinct prerequisites

The customer has to provide one of two identity solutions:
- **Either** we require a configured [_'Native' **ADDS**_](#Native-AD-prerequisites) (AD, Active Directory) environment to enable an authentication with the file share. This can be achieved either via deploying Domain Controllers to the cloud or 'just' extending the on-premises domain by allowing network level access to the Domain Controllers from the WVD subnets (via VPN or ExpressRoute)
- **or** an [_'Azure' **ADDS**_](#Azure-ADDS-Prerequisites) domain.

## Native AD prerequisites
> ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x)

<div style="padding: 10px; border-left: 1px solid #FFC000;" >

   ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) **NOTE**: Only required for Native-AD authentication

The following prerequisites are the ones outlined in yellow in the above diagram and having a small DC icon on the top right corner .

### Identity Solution

- ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) ![Icon-identity-221-Azure-Active-Directory.svg](/.attachments/Icon-identity-221-Azure-Active-Directory-647d4514-695c-43e0-8137-c336445e4388.svg =15x) _**Native AD synced to Azure AD**_

   A Windows Server Active Directory in sync with Azure Active Directory. This can be configured with Azure AD Connect.

### Native-AD Users

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**Domain Join Service Account**_

  This is the Native AD service account with sufficient privileges to join machines to the domain. The domain join of the WVD host pool's VMs is performed on behalf of this account as part of the WVD host pool deployment. 

### Native-AD Groups

- ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) _**WVD User Group(s)**_

    This are one or more AD groups synced to Azure AD designed to host the designated WVD users. On a later stage, these groups are assigned to the _Application Groups_ to enable access to the WVD services and optionally enabled on the FSLogix profile file shares to enable the user context synchronization.

<div style="padding: 10px; border-left: 1px solid #ED7D31;" >

![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) **NOTE:** Only required in case the FSLogix user data synchronization is deployed

### Additional prerequisites for FSLogix 

In this scenario, all **AD identities** used to access Azure file shares must be **synced to Azure AD** to enforce share level file permissions through the standard role-based access control (RBAC) model.

 > **Note**: For further details or possible updates on this preview, please refer [here](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-auth-active-directory-enable#regional-availability).  <span style="color:red">[<TODO>: @<603DE783-D07C-6BF7-8416-0B68F35E94EB>, is this still in preview?]</span>

- ![Icon-compute-21-Virtual-Machine.svg](/.attachments/Icon-compute-21-Virtual-Machine-347d68cb-caf8-41aa-a24e-a13d5dfa3ef6.svg =15x) _**Domain Joined machine**_

  The steps within the '_AD authentication enablement for Azure Files_' phase must be performed on a device that is domain joined to AD. That can be an on-premises machine or an Azure VM.
This machine is leveraged by the _**Subscription & AD Admin**_ to enable Active Directory Domain Service for authentication to Azure file shares. Connectivity to Azure is required.

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**Subscription & AD Admin**_

  The steps within the '_AD authentication enablement for Azure Files_' phase must be performed on behalf of a Native AD User synced to Azure AD who holds the following:

  - Required Permissions (Azure)

    | Scope| Configuration | Role |
    |--|--|--|
    | Subscription | RBAC (IAM) | `Contributor` |
    
  - Required Permissions (AD)
Create a service logon account or a computer account in the target AD.

</div>

</div>

---

## Azure ADDS prerequisites
> ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x)

<div style="padding: 10px; border-left: 1px solid #85DFFF;" >
 
   ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x)  **NOTE**: Only required for Azure-ADDS authentication

The following prerequisites are the ones outlined in light blue in the above diagram and having a small Azure ADDS icon on the top right corner.

> **NOTE**: this scenario also utilizes a couple of services users, but these are programmatically configured during the deployment phase.

### Identity Solution
  - ![azure-logo.png](/.attachments/azure-logo-df90809a-7022-452d-b5e5-4f0d777dc45b.png =15x) ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x) _**Azure ADDS**_ 

    If the customer wants to leverage _Azure ADDS_ for the solution, it must be fully set up and configured prior to the WVD project delivery begins.

### Active-AD Users

- ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) _**Domain Join Service Account**_

  This is the Azure AD service account with sufficient privileges to join machines to the domain. The domain join of the WVD host pool's VMs is performed on behalf of this account as part of the WVD host pool deployment. 

### Active-AD Groups

- ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) _**WVD User Group(s)**_

    This are one or more Azure AD groups designed to host the designated WVD users. On a later stage, these groups are assigned to the _Application Groups_ to enable access to the WVD services and optionally enabled on the FSLogix profile file shares to enable the user context synchronization.

</div>

---

**Note**: For additional general WVD requirements, such as supported remote desktop clients and OS images, please refer to the official documentation [here](https://docs.microsoft.com/en-us/azure/virtual-desktop/overview#requirements)