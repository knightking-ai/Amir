This deployment approach leverages Azure DevOps pipelines (Pipeline-Orchestration) and a Git Repository. 

Before using the pipelines to deploy the WVD solution, several prerequisites must be met. The required steps may vary, as they depend on the customer's identity solution (Azure ADDS or Native Active Directory).

This section provides an overview of the deployment concept for WVD automation.

[[_TOC_]]

---

# Overview

The deployment has two main phases: 

![overview.png](/.attachments/overview-04078d0a-0ebf-4f10-85f3-50bbc68af318.png =75%x)

Pre-Deployment setup
- Prerequisites: a number of infrastructure components must be set up by the customer.
- DevOps Enablement: an Azure DevOps project has to be prepared for using this automated solution.
- WDV Service Enablement: the WVD service has to be onboarded by the customer's AAD Global Administrator to their Azure AD Tenant.

Deployment
- Resource Deployment: the main deployment of this automated solution (depending on the scenario, it may contains certain manual steps)

Each step is described in greater details in this section's subpages.

![wvdFull.png](/.attachments/wvdFull-0fa4801f-8c50-4c86-aa64-3f65e38c7bd7.png)

# Glossary
|| **Item**                                       | **Type**                | **Description** |
|-| ------------------------------------------ | ------------------- | ----------- |
| ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x) | AADDS                                      | Azure Service       | The '_Azure Active Directory Domain Services_' is used as an extension of the Azure AD tenant and leveraged for authentication of WVD users towards the File-Share to synchronize profiles. |
| ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) | Native AD synced to Azure AD | Hybrid | A Windows Server Active Directory in sync with Azure Active Directory. This can be configured with Azure AD Connect. |
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Components Storage Account                 | Azure Service       | The '_Components_' storage account is used as an ARM template store and referenced by the deployment pipeline(s). It follows the structure of Infrastructure as Code approach in the ACF offer, where tested ARM templates are uploaded to a storage account to enable a pre-validated consumption of changes. Though the validation pipelines of each module also upload the modules to the components storage account after the successful tests, it is also possible to manually upload an initial set of templates used in previous deployments to get up to speed faster. |
| ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) | Deployment Service Principal               | Application         | The service principal used for the Azure DevOps service connection. This service principal is used by the pipeline to deploy all required resources. |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Domain Join Service Account                        | Azure AD User             | _**AADDS approach**_: This is the user performing the domain-join of WVD machines within the _Host Pool_ deployment. The required '_Domain Join Service Accounts_' credentials are stored in a key vault secret prior to the _Host Pool_ deployment for securing their access. <br /> A dedicated user with required domain-join privileges and corresponding secret are created in the pipeline. |
|  ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Domain Join Service Account | AD User in sync with Azure AD           | _**Native AD approach**_: This is the user performing the domain-join of WVD machines within the _Host Pool_ deployment. The required '_Domain Join Service Accounts_' credentials are stored in a key vault secret prior to the _WVD Host Pool_ deployment for securing their access. <br /> A user with required domain-join privileges must be already in place in the customer environment. The corresponding password is stored as a key vault secret within the key vault and storage account configuration step. |
| ![Icon-compute-21-Virtual-Machine.svg](/.attachments/Icon-compute-21-Virtual-Machine-347d68cb-caf8-41aa-a24e-a13d5dfa3ef6.svg =15x) | Domain Joined Machine | On-premises machine <br />_**-OR-**_ Azure VM | Must be domain-joined to AD (also referred as AD DS) and must have connectivity to Azure. This machine will be used by the '_WVD AD Auth Admin_' to enable Active Directory Domain Service for authentication to Azure file shares. |
| ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x) | File Share Access Group                    | Azure AD Group            | _**AADDS approach**_ This group contains WVD users who need access to the file share in the storage account that stores user profiles and configuration scripts. Users in this group will hold the '_Storage File Data SMB Share Contributor_' RBAC role on the storage account. <br /> This group is created in the pipeline.  |
| ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) | File Share Access Group           | AD Group in sync with Azure AD           | _**Native AD approach**_ This group contains WVD users who need access to the file share in the storage account that stores user profiles and configuration scripts. Users in this group will hold the '_Storage File Data SMB Share Contributor_' RBAC role on the storage account. <br /> This group must be already in place in the customer environment.  |
| ![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) | FSLogix | Tool | Used to perform the synchronization of user profile in between configured VMs and a configured file share. ([Docs](https://docs.microsoft.com/en-us/fslogix/overview)) |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Global Administrator                       | Azure AD User             | An AAD global administrator user should be set up in the customer's environment. With this user, we can set up the RDS tenant and create all required resources required for later deployments. |
| ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =15x) | Key Vault                                  | Azure Service       | A key vault is created to store the secrets of the identities to be used, i.e. the '_Tenant Admin Service Principal_' and '_Domain Join Service Account'. |
| ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) | Key Vault Access Group                     | Azure AD Group            | The key vault access group is created and assigned an access policy to grant members access to the key vault's secrets. The primary member of this group is the '_Deployment Service Principal_' as it must be able to access the required secrets for the _WVD Host Pool_ deployment. |
| ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =15x) | RDS Tenant                                 | Tenant              | The RDS tenant is created by the '_Global Administrator_' and represents the environment for any WVD registered session hosts, app groups, users, etc. | 
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Storage Account                            | Azure Service       | This storage account has two main functions: First, it holds all required files necessary for any host pool deployment, that is the custom script extension resources. Second, its file share is used to synchronize user profiles across machines using FSLogix. As such, it needs the AADDS authentication enabled. |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x)  | Subscription & AAD Admin | Azure AD User | This user will store the Native AD '_Domain Join Service Account_' password in a key vault secret. |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x)  | Subscription & AD Admin | Native AD domain user in sync with Azure AD | This user will enable the recently introduced (public preview) support of leveraging Active Directory Domain Service for authentication to Azure file shares. |
| ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) | Tenant Admin Service Principal             | Application         | This service principal is assigned the 'RDS Owner' role for the given RDS tenant by the '_Global Administrator_'. Using this service principal, the WVD Host Pool deployment can run a DSC extension and register the nodes on the tenant as sessions hosts. Furthermore, operations team members maintaining the RDS tenant can use this service principal to perform configurations. |
| ![Icon-networking-61-Virtual-Networks.svg](/.attachments/Icon-networking-61-Virtual-Networks-527ab1d3-5066-45fb-a793-0bc3430dcb16.svg =15x) | VNET                                       | Azure Service       | A VNET and contained subnet must be provided to deploy the _Host Pools_ into. As such, it must be large enough to host the requested amount of sessions hosts. |
| ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) | Windows Virtual Desktop Application        | Managed Application | This managed application is registered by the '_Global Administrator_' on the customer's AAD tenant and acts as a baseline for the RDS tenant. On it, the '_Global Administrator_' holds the 'TenantCreator' AppRole. |
| ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) | Windows Virtual Desktop Client Application | Managed Application | This managed application is registered by the '_Global Administrator_' on the customer's AAD tenant and acts as a baseline for the RDS tenant. |
| ![wvd.png](/.attachments/wvd-383d1d43-8010-4094-9398-a1f175acd142.png =15x) | WVD Host Pool                              | Azure Service       | The _WVD Host Pool_ encapsulates the nodes users can log into or run remote applications on. |

> **Assumptions**:
> - Naming convention for Azure has been defined

# Deployment Paths
If you are planning to use FSLogix as a profile management solution for your deployment, there are two options you can take.

>**NOTE**: This solution currently only supports Azure Files as the storage option (NetApp Files and Storage Spaces Direct support may be added later).

## 'Native AD' file share authentication
![wvdFullAd.png](/.attachments/wvdFullAd-c155f4ea-9894-4362-825a-77eb6731ee4c.png =70%x)

## 'Azure Active Directory Domain Service' file share authentication
![wvdFullAADDS.png](/.attachments/wvdFullAADDS-9dc41754-88f0-47e4-8dde-fb380c3250e5.png =70%x)