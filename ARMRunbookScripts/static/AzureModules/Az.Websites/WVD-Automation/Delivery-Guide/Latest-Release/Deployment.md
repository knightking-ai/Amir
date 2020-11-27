This deployment approach leverages Azure DevOps pipelines (Pipeline-Orchestration) and a Git Repository. 

Before using the pipelines to deploy the WVD solution, several prerequisites must be met. The required steps may vary, as they depend on the customer's identity solution (Azure ADDS or Native Active Directory).

This section provides an overview of the deployment concept for WVD automation.

[[_TOC_]]

---

# Overview

The deployment has two main phases: 

![wvdoverview.png](/.attachments/wvdoverview-f35df0e3-2dd1-4d8d-a91e-63ce6e6d959b.png =50%x)

Pre-Deployment setup
- Prerequisites: a number of infrastructure and AD components must be set up by the customer.
- DevOps Enablement: an Azure DevOps project has to be prepared for using this automated solution.

Deployment
- Resource Deployment: the main deployment of this automated solution (depending on the scenario, it may contains certain manual steps)

Each step is described in greater details in this section's subpages.

![wvdProcessOverview.png](/.attachments/wvdProcessOverview-c1cc6b5f-fc72-450f-9ff9-e15ec2fb15c5.png)

---
---

# Glossary

Following, you can find an overview of the most important elements of the DevOps automation solution. The _'Required for'_ column is to be read as _'Required if using feature'_ 

## Prerequisites
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![Icon-networking-61-Virtual-Networks.svg](/.attachments/Icon-networking-61-Virtual-Networks-527ab1d3-5066-45fb-a793-0bc3430dcb16.svg =15x) | VNET | VNET | | A VNET and contained subnet must be provided to deploy the _Host Pools_ into. As such, it must be large enough to host the requested amount of sessions hosts. |
| ![Icon-identity-222-Azure-AD-Domain-Services.svg](/.attachments/Icon-identity-222-Azure-AD-Domain-Services-b89a6dd6-a5b7-4241-bdc7-849ef677d01c.svg =15x) | AADDS | AADDS | AADDS authentication | The '_Azure Active Directory Domain Services_' is used as an extension of the Azure AD tenant and leveraged for authentication of WVD users towards the File-Share to synchronize profiles. |
| ![AD.png](/.attachments/AD-0a305357-773c-4be7-a11e-7736e742703a.png =15x) | Native-AD | Native-AD synced to Azure AD | Native-AD authentication| A Windows Server Active Directory in sync with Azure Active Directory. This can be configured with Azure AD Connect. |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Azure AD User | Domain Join Service Account | AADDS authentication | This is the user performing the domain-join of WVD machines within the _Host Pool_ deployment. The required '_Domain Join Service Account_' credentials are stored in a key vault secret prior to the _Host Pool_ deployment for securing their access. <br /> A dedicated user with required domain-join privileges and corresponding secret are created in the pipeline. |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Native AD User <br />_**-OR-**_ AD User in sync with Azure AD | Domain Join Service Account | Native-AD authentication | This is the user performing the domain-join of WVD machines within the _Host Pool_ deployment. The required '_Domain Join Service Account_' credentials are stored in a key vault secret prior to the _Host Pool_ deployment for securing their access. <br /> A service account with required domain-join privileges must be already in place in the customer environment. Also, the '_Domain Join Service Account Secret_' containing the corresponding credential, must be securely stored in a DevOps secret prior to the deployment. |
| ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) | Azure AD Group | WVD User Group | Native-AD authentication | This group contains WVD users who need access to the file share in the storage account that stores user profiles and configuration scripts. Users in this group will hold the '_Storage File Data SMB Share Contributor_' RBAC role on the storage account. <br /> This group is created by the pipeline within the storage account post deployment. |
| ![Icon-identity-223-Groups.svg](/.attachments/Icon-identity-223-Groups-41ab1233-47b1-41da-bf05-f89cf18f2934.svg =15x) | AD Group in sync with Azure AD | WVD User Group | Native-AD authentication | This group contains WVD users who need access to the file share in the storage account that stores user profiles and configuration scripts. Users in this group will hold the '_Storage File Data SMB Share Contributor_' RBAC role on the storage account. <br /> This group must be already in place in the customer environment. |
| ![Icon-compute-21-Virtual-Machine.svg](/.attachments/Icon-compute-21-Virtual-Machine-347d68cb-caf8-41aa-a24e-a13d5dfa3ef6.svg =15x) | Domain Joined Machine | On-premises machine <br />_**-OR-**_ Azure VM | Native-AD authentication <p> FSLogix Profile Sync | Must be domain-joined to AD (also referred as AD DS) and must have connectivity to Azure. This machine will be used by the '_Subscription & AD Admin_' to enable Active Directory Domain Service for authentication to Azure file shares (Preview). |
| ![Icon-identity-230-Users.svg](/.attachments/Icon-identity-230-Users-042cb04a-e97f-4918-ad6b-77192250d623.svg =15x) | Native AD domain user in sync with Azure AD | Subscription & AD Admin | Native-AD authentication <p> FSLogix Profile Sync | This user will enable the recently introduced (public preview) support of leveraging Active Directory Domain Service for authentication to Azure file shares. |

---

## DevOps Enablement
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![Icon-devops-261-Azure-DevOps.png](/.attachments/Icon-devops-261-Azure-DevOps-fd7498ae-bba7-40dd-8000-5d96307bdd50.png =15x) | SaaS | Azure DevOps | | The DevOps project to host the solution repository, variables and pipelines. |
| ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-9dc15c8a-0738-416e-82bd-36ac7cfaac92.svg =15x) | AzureAD Application | Deployment Service Principal | | The service principal used for the Azure DevOps service connection. This service principal is used by the pipeline to deploy all required resources. |
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Storage Account | Components Storage Account | | The '_Components_' storage account is used as an ARM template store and referenced by the deployment pipeline(s). It follows the structure of Infrastructure as Code approach in the ACF offer, where tested ARM templates are uploaded to a storage account to enable a pre-validated consumption of changes. Though the validation pipelines of each module also upload the modules to the components storage account after the successful tests, it is also possible to manually upload an initial set of templates used in previous deployments to get up to speed faster. |
| ![secret.png](/.attachments/secret-295d7265-7816-4bbb-a2b3-d32c7ad81e94.png =15x) | DevOps variable | Domain Join Service Account Password| | The '_Domain Join Service Accounts_' password is stored as a secret variable in DevOps as part of the DevOps enablement. This must be done prior to the deployment for securing access to this password. The corresponding value is gathered from DevOps and stored in a key vault secret by the pipeline within the key vault post deployment. |
| ![secret.png](/.attachments/secret-295d7265-7816-4bbb-a2b3-d32c7ad81e94.png =15x) | DevOps variable | Local Admin Password| | Can be provided as a local admin password for the WVD session host VMs. If no value is provided, the Mgmt-RG deployment will generate a password and store it in the Mgmt-RGs key vault. |

---

## Resource Deployment
### Profile Resource Group
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Storage Account | Storage Account |  FSLogix Profile Sync | Storage account file shares are used to synchronize user profiles across machines using FSLogix. As such, it needs the NativeAD/AADDS authentication enabled. |
| ![Icon-storage-400-Azure-Fileshare.png](/.attachments/Icon-storage-400-Azure-Fileshare-c17002f0-a33f-46e8-b014-185e3ef9c6e5.png =15x) | Storage Account File Share | File Share | FSLogix Profile Sync | Hosts the user data synced by FSLogix. |
| ![rbac.png](/.attachments/rbac-c74dc4b3-d9aa-4ed2-a9a9-abf9c7812751.png =15x) | Role Assignment | SMB Contributor [WVD User Group] | FSLogix Profile Sync | One of the requirements to be met to enable the user data sychronization. |

### Management Resource Group
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =15x) | Key Vault | Key Vault | | A key vault is created to store several secrets used during the deployments. i.e. the '_Domain Join Service Account'. |
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Storage Account | Storage Account | | This storage account has several containers with the files necessary for several deployments like the Host-Pool deployment CSE, or Automation Account Runbook scaling script. |
| ![Recovery Services Vaults.svg](/.attachments/Recovery%20Services%20Vaults-3297985c-7e03-4ed2-8f7e-d2845d5e7a2e.svg =15x) | Recovery Services Vault | Profile Backup Vault | FSLogix Profile Backup | Can be deployed to backup the File-Shares populated by FSLogix. |
| ![Icon-manage-322-Automation-Accounts.svg](/.attachments/Icon-manage-322-Automation-Accounts-18077721-1b49-42d4-b5bd-9b1a8f0becf7.svg =15x) | Automation Account | Scaling Automation Account | HostPool scaling | Contains the scaling runbook and is triggered by the 'WVD Scaling Scheduler(s)'. Responsible for scaling any given host pool. |

### Host-Pool Resource Group
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![wvdHostPool.png](/.attachments/wvdHostPool-05d32ba8-e027-44c5-804f-4612e5d889c7.png =15x) | WVD Host Pool | Host Pool | | A central resource to link several WVD resources like the session hosts and application groups together. |
| ![wvdApplicationGroup.png](/.attachments/wvdApplicationGroup-63915ba5-490b-4567-8474-0eaf1039d436.png =15x) | WVD Application Group | Application Group | | Can be both a Desktop-Application-Group for desktop users and Remote-Application-Group for application users. |
| ![vm.png](/.attachments/vm-0c755e3b-6069-4e14-a221-2d3056a09000.png =15x) | WVD Virtual Machine | Session Host | | VMs that carry the WVD workloads (e.g. user sessions). |
| ![wvdWorkspace.png](/.attachments/wvdWorkspace-e4f6a091-9334-4577-9311-cf910a96b8b6.png =15x) | WVD Workspace | Workspace | | Application Groups linked to a workspace show up as published in the WVD client. |
| ![wvdApplication.png](/.attachments/wvdApplication-6aee1e71-a10b-42d4-a742-cc582679b5ed.png =15x) | WVD Remote Application | Remote App | | Links applications that are installed on the session host(s) to a Remote-Application-Group. |
| ![Logic Apps.svg](/.attachments/Logic%20Apps-fd808d7d-e2e5-4b69-bbf0-f4abe34b883e.svg =15x) | Logic App Workflow | Scaling Scheduler | HostPool scaling | A Logic-App-Workflow deployed for each Host-Pool to trigger the host pool scaling. |

### Imaging Resource Group
|  | **Type** | **Name** | **Required for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![managedServiceIdentity.png](/.attachments/managedServiceIdentity-ddc29c39-2b03-42b4-b810-82aead05331b.png =15x) | Managed Service Identity | Imaging MSI | Image Lifecyle | |
| ![imageGallery.png](/.attachments/imageGallery-19e9e059-582a-4910-8ef3-b0b1354781d4.png =15x)  | Shared Image Gallery | Image Gallery | Image Lifecyle | Image versioning and world-wide replication service |
| ![rbac.png](/.attachments/rbac-c74dc4b3-d9aa-4ed2-a9a9-abf9c7812751.png =15x) | Role Assignment | MSI Subscription Contributor | Image Lifecyle | |
| ![imageDefinition.png](/.attachments/imageDefinition-84e9fcc5-3395-4af6-a5ea-6f48d0bc713a.png =15x) | Image Definition | Shared Image Definition | Image Lifecyle |
| ![imageTemplate.png](/.attachments/imageTemplate-ae177fb7-83ed-4671-8627-b0257bfed861.png =15x) | Image Template | Image Template | Image Lifecyle | |

---

## Others
|  | **Type** | **Name** | **Required only for** | **Description** |
|--|----------|----------|-----------------------|-----------------|
| ![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) | Software | FSLogix | FSLogix Profile Sync | Used to perform the synchronization of user profile in between configured VMs and a configured file share. ([Docs](https://docs.microsoft.com/en-us/fslogix/overview)) |

> **Assumptions**:
> - Naming convention for Azure has been defined