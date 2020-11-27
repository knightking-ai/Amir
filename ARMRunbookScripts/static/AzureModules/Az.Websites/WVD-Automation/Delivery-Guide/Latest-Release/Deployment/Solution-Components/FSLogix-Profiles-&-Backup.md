_'FSLogix'_ takes care of the user data synchronization in between sessions hosts and any given file share. A recovery services vault can be used to further secure the data as part of a recurring backup.

[[_TOC_]]

<div style="padding: 10px; border-left: 1px solid #ED7D31;" >

![fslogixIcon.png](/.attachments/fslogixIcon-d387b2f6-7b3d-4627-be6f-c15f4161561d.png =15x) **NOTE:** Only required in case the FSLogix user data synchronization is deployed


# FSLogix
 <span style="color:red">[<TODO>: Hola @<603DE783-D07C-6BF7-8416-0B68F35E94EB>]</span>

---
---

# Backup
As an additional layer of security on top of the provided FSLogix solution it is recommended to deploy a recovery services vault to backup the deployed Azure Files. As described in initial table, this resource should be deployed in the Management ResourceGroup and is therefore part of its pipeline.

## Components
The feature involves two major components:

| | Resource | Resource Group | Description |
|--|--|--|--|
| ![Icon-storage-400-Azure-Fileshare.svg](/.attachments/Icon-storage-400-Azure-Fileshare-15ecf767-89da-47ae-986d-3e6d7b67a883.svg =15x) | File Share (Storage Account) | WVD-Profiles-RG | Hosts the user data synced by FSLogix. |
| ![Recovery Services Vaults.svg](/.attachments/Recovery%20Services%20Vaults-3297985c-7e03-4ed2-8f7e-d2845d5e7a2e.svg =15x) | Recovery Services Vault | WVD-Mgmt-RG | Can be deployed to backup the File-Shares populated by FSLogix. |

---

## Process
The process itself is carried out in three steps: 
1. The pipeline deployes the RSV resources alongside a BackupPolicy and links the required file share storage accounts back to the RSV (Note: A storage account can be linked to a single RSV at any given time only)
2. A post-deployment script registers the actual file shares associated with these storage accounts as _Backup Items_ in the RSV
3. The backup policy kicks in and maintains the backup basted on its configuration

      ![rsvFlow.png](/.attachments/rsvFlow-78466cbc-a328-489a-8bf7-da2716fe66ee.png =400x)

---

## Configuration

When deploying the feature, the main aspects to clarify are
- **Which file shares must be considered?**
  
   The answer to this question defines which file shares must be added to the Mgmt-ResourceGroup [RSV deployment configuration](/Delivery-Guide/Latest-Release/Deployment/Resource-Deployment/Management-Resource-Group#profiles-recovery-services-vault-(rsv))

- **How should the backup be configured?**
  
   This question relates tohow the _Backup Policy_ linked to the backup items should be configured. By default, the solution provides an example policy _'filesharepolicy'_. 

</div>