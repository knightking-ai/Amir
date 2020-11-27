This section describes the individual components deployed into the Imaging Resource Group_, when they are needed and what parameters are required for each.

[[_TOC_]]

<span style="color:red">[<TODO>: ADD Imaging IMAGE]</span>

---

# Deployments

## Imaging Managed Service Identity
> ![managedServiceIdentity.png](/.attachments/managedServiceIdentity-d20669e9-966c-499d-a8a0-6da83ed43ca9.png =25x)

Azure Active Directory feature that eliminates the need for credentials in code, rotates credentials automatically, and reduces identity maintenance. In the context of the imaging solution, the MSI is used by the Image Builder Service as well the pipeline to trigger the image template creation.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| userMsiName | string | `[guid(resourceGroup().id)` | - | Optional |Name of the User Assigned Identity.

### Required Variables
_None_

### Custom Tasks
_None_

---

## Shared Image Gallery
> ![imageGallery.png](/.attachments/imageGallery-33deba72-0914-41de-b573-ca1a8519d232.png =25x)

Azure service that helps to build structure and organization for managed images. Provides global replication, versioning and grouping, and sharing across subscriptions, and scaling.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `galleryName` | string | - | - | Required | Name of the Azure Shared Image Gallery
| `galleryDescription` | string | - | - | Optional | Description of the Azure Shared Image Gallery |

### Required Variables
_None_

### Custom Tasks
_None_

---

## Role Assignment
> ![rbac.png](/.attachments/rbac-7d3a0e8a-bd6a-4478-a654-938612340f80.png =15x)

Assign the _'Subscription Contributor'_ role to the deployed MSI.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `roleAssignments` | array | [] | - | Optional | Array of role assignment objects that contain the 'roleDefinitionIdOrName' and 'principalIds' to define RBAC role assignments on this resource. In the roleDefinitionIdOrName attribute, you can provide either the display name of the role definition, or it's fully qualified ID in the following format: '/providers/Microsoft.Authorization/roleDefinitions/c2f4ef07-c644-48eb-af81-4b1b4947fb11' |  

### Required Variables
_None_

### Custom Tasks
_None_

---

## Image Definition
> ![imageDefinition.png](/.attachments/imageDefinition-831f012b-23c4-451c-ace2-9353b87ce703.png =25x)

Created within a gallery and hold information about the image and requirements for using it internally. This includes whether the image is Windows or Linux, release notes, and minimum and maximum memory requirements.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `imageDefinitionName` | string | - | - | Required | Name of the image definition.
| `galleryName` | string | - | - | Required | Name of the Azure Shared Image Gallery
| `osType` | string | `Windows` | `Windows` or `Linux` | Optional | OS type of the image to be created.
| `osState` | string | `Generalized` | `Generalized` or `Specialized` | Optional | This property allows the user to specify whether the virtual machines created under this image are 'Generalized' or 'Specialized'.
| `publisher` | string | `MicrosoftWindowsServer` | - | Optional | The name of the gallery Image Definition publisher.
| `offer` | string | `WindowsServer` | - | Optional | The name of the gallery Image Definition offer.
| `sku` | string | `2019-Datacenter` | - | Optional | The name of the gallery Image Definition SKU.
| `minRecommendedvCPUs` | int | `1` | `1-128` | Optional | The minimum number of the CPU cores recommended for this image.
| `maxRecommendedvCPUs` | int | `4` | `1-128` | Optional | The maximum number of the CPU cores recommended for this image.
| `minRecommendedMemory` | int | `4` | `1-4000` | Optional | The minimum amount of RAM in GB recommended for this image.
| `maxRecommendedMemory` | int | `16` | `1-4000` | Optional | The maximum amount of RAM in GB recommended for this image.
| `hyperVGeneration` | string | `V1` | `V1` or `V2` | Optional | The hypervisor generation of the Virtual Machine. Applicable to OS disks only. - V1 or V2

### Required Variables
_None_

### Custom Tasks
_None_

---

## Image Template
> ![imageTemplate.png](/.attachments/imageTemplate-40b6635a-2b8c-4f44-b0c7-7d203d9d1086.png =25x)

A standard Azure Image Builder template that defines the parameters for building a custom image with AIB. The parameters include image source (Marketplace, custom image, etc), customization options (i.e., Updates, scripts, restarts), and distribution (i.e., managed image, shared image gallery).

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `imageTemplateName` | string | - | - | Required | Name of the Image Template to be built by the Azure Image Builder service.
| `imageSource` | object | - | - | Required | Image source definition in object format.
| `customizationSteps` | array | - | - | Required | Customization steps to be run when building the VM image.
| `userMsiName` | string | - | - | Required | Name of the User Assigned Identity to be used to deploy Image Templates in Azure Image Builder.
| `vmSize` | string | `Standard_D2s_v3` | - | Optional | Specifies the size for the VM.
| `osDiskSizeGB` | int | `127` | - | Optional | Specifies the size of OS disk.
| `subnetId` | string | - | - | Optional | Resource Id of an already existing subnet, e.g. `/subscriptions/<subscriptionId>/resourceGroups/<resourceGroupName>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<subnetName>`. If no value is provided, a new VNET will be created in the target Resource Group.
| `managedImageName` | string | - | - | Optional | Name of the managed image that will be created in the AIB resourcegroup.

### Required Variables
| Variable Name | Type | Description |
| :- | :-  | :- | :- |
| `ResourceGroupName` | string | Resource group to create the image in |
| `ImageTemplateName` | string | Name of the image template |

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Trigger Image Build**
   - **_Description:_** Once the image template resource is created, we can trigger the image build using the subsequent script. It is recommended to run it as part of a post-deployment task in the pipeline.
   - **_Script:_** `Invoke-AzResourceAction.ps1` 