This section describes the individual components deployed into the _Host-Pool Resource Group_, when they are needed and what parameters are required for each.

[[_TOC_]]

<span style="color:red">[<TODO>: ADD Host-Pool IMAGE]</span>

---

# Deployments

## WVD Host Pool
> ![wvdHostPool.png](/.attachments/wvdHostPool-05d32ba8-e027-44c5-804f-4612e5d889c7.png =25x)

A central resource to link several WVD resources like the session hosts and application groups together. 

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `hostpoolName` | string | - | - | Required | Name of the Host Pool | 
| `hostpoolFriendlyName` | string | - | - | Optional | The friendly name of the Host Pool to be created. |
| `hostpoolDescription` | string | - | - | Optional | The description of the Host Pool to be created.
| `hostpoolType` | string | `Pooled` | "Personal", "Pooled" | Optional | Set this parameter to Personal if you would like to enable Persistent Desktop experience. |
| `personalDesktopAssignmentType` | string | `Automatic` | "Automatic", "Direct", "" | Optional | Set the type of assignment for a Personal Host Pool type |
| `maxSessionLimit` | int | `9999` | - | Optional| Maximum number of sessions. |
| `loadBalancerType` | string | `true` | "BreadthFirst", "DepthFirst", "Persistent" | Optional | Type of load balancer algorithm.

### Required Variables
_None_

### Custom Tasks
_None_

---

## WVD Application Group
> ![wvdApplicationGroup.png](/.attachments/wvdApplicationGroup-63915ba5-490b-4567-8474-0eaf1039d436.png =25x)

Can be both a Desktop-Application-Group for desktop users and Remote-Application-Group for application users.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `appGroupName` | string | - | - | Required | Name of the Application Group to create this application in |
| `appGroupType` | string | - | "RemoteApp", "Desktop" | Required | Type of application group
| `hostpoolName` | string | - | - |  Required | Name of the Host Pool to be linked to this Application Group. |
| `appGroupFriendlyName` | string | - | - |  Optional | The friendly name of the Application Group to be created |
| `appGroupDescription` | string | - | - | Optional | The description of the Application Group to be created. |
| `roleAssignments` | array | [] | - | Optional | Array of role assignment objects that contain the 'roleDefinitionIdOrName' and 'principalIds' to define RBAC role assignments on this resource. In the roleDefinitionIdOrName attribute, you can provide either the display name of the role definition, or it's fully qualified ID in the following format: '/providers/Microsoft.Authorization/roleDefinitions/c2f4ef07-c644-48eb-af81-4b1b4947fb11' |

### Required Variables
_None_

### Custom Tasks
_None_

---

## WVD Session Host
> ![vm.png](/.attachments/vm-0c755e3b-6069-4e14-a221-2d3056a09000.png =25x)

VMs that carry the WVD workloads (e.g. user sessions).

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `vmNamePrefix` | string | [take(toLower(uniqueString(resourceGroup().name)),10)] | - | Optional | If no explicit values were provided in the vmNames parameter, this prefix will be used in combination with the vmNumberOfInstances and the vmInitialNumber parameters to create unique VM names. You should use a unique prefix to reduce name collisions in Active Directory. If no value is provided, a 10 character long unique string will be generated based on the Resource Group's name. |
| `vmNumberOfInstances` | int | 1 | 1-800 | Optional | If no explicit values were provided in the vmNames parameter, this parameter will be used to generate VM names, using the vmNamePrefix and the vmInitialNumber values | 
| `vmInitialNumber` | int | 1 | - | Optional | If no explicit values were provided in the vmNames parameter, this parameter will be used to generate VM names, using the vmNamePrefix and the vmNumberOfInstances values |
| `vmSize` | string | `Standard_D2s_v3` | - | Optional | Specifies the size for the VMs | 
| `imageReference` | object | - | - | Optional | OS image reference. In case of marketplace images, it's the combination of the publisher, offer, sku, version attributes. In case of custom images it's the resource ID of the custom image. |
| `adminUsername` | securestring | - | - | Required | Administrator username.|
| `adminPassword` | securestring| - | - | Optional | When specifying a Windows Virtual Machine, this value should be passed. |
| `availabilitySetName`| string| - | - | Optional | Creates an availability set with the given name and adds the VMs to it. Cannot be used in combination with availability zone nor scale set.|
| `subnetId`| string| - | - | Required | Full qualified subnet Id. |
| `domainName`| string| - | - | Optional | Specifies the FQDN the of the domain the VM will be joined to. Currently implemented for Windows VMs only. |
| `domainJoinUser`| string | - | - | Mandatory if domainName is specified | User used for the Domain join operation. Format: username@domainFQDN.|
| `domainJoinPassword`| Secure String | - | - | Mandatory if domainName is specified | Password of the user specified in domainJoinUser parameter|
| `domainJoinOU`| string| - | - | Optional | OU where to store the computer account for the domain joined |
| `dscConfiguration`| object| - | - | Optional | The DSC configuration object |

### Required Variables
_None_

### Custom Tasks
_None_

---

## WVD Workspace
> ![wvdWorkspace.png](/.attachments/wvdWorkspace-e4f6a091-9334-4577-9311-cf910a96b8b6.png =25x)

Application Groups linked to a workspace show up as published in the WVD client.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `workspaceName` | string | - | - | Required | The name of the Workspace to be attach to new Application Group |
| `appGroupResourceIds` | string | [] | - | Required | Resource IDs fo the existing Application groups this workspace will group together | 
| `workspaceFriendlyName` | string | - | - | Optional | The friendly name of the Workspace to be created |
| `workspaceDescription` | string | - | - | Optional | The description of the Workspace to be created | 

### Required Variables
_None_

### Custom Tasks
_None_

---

## WVD Remote Application
> ![wvdApplication.png](/.attachments/wvdApplication-6aee1e71-a10b-42d4-a742-cc582679b5ed.png =25x)

Links applications that are installed on the session host(s) to a Remote-Application-Group.

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `applications` | array | [] | Complex structure, see below. | Required | List of applications to be created in the Application Group. |
| `appGroupName` | string | - | - | Required | Name of the Application Group to create the application(s) in. |
| `cuaId` | string | - | - | Optional | Customer Usage Attribution id (GUID). This GUID must be previously registered |

### Required Variables
_None_

### Custom Tasks
_None_

---

## WVD Scaling Scheduler
> ![Logic Apps.svg](/.attachments/Logic%20Apps-fd808d7d-e2e5-4b69-bbf0-f4abe34b883e.svg =25x)

A Logic-App-Workflow deployed for each Host-Pool to trigger the host pool scaling. 

### Related Deployments
_None_

### Required Parameters
| Parameter Name | Type | Default Value | Possible values | Mode | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `logicAppName` | string | - | - | Required | The name of the logic app to create |
| `webhookURI` | string | - | - | Required | Webhook URI of Logic App |
| `recurrenceInterval` | int | - | - | Required | Specifies the recurrence interval of the job in minutes |
| `actionSettingsBody` | object | - | - | Required | Specifies the body in Action settings ('Note': Input should be in json format) |

### Required Variables
_None_

### Custom Tasks
_None_

---

## Session Host Image Update
> ![imageTemplate.png](/.attachments/imageTemplate-40b6635a-2b8c-4f44-b0c7-7d203d9d1086.png =25x)

This deployment takes care of the VMs that run a deprecated VM image. This includes notifying current users, as well as shutting down (or permanenty deleting) these VMs where possible.

### Related Deployments
_None_

### Required Parameters
_None_

### Required Variables
| VariableName | Type | Description |
| :-             | :-   | :-            | :-              | :-   | :-          |
| `` | | | | | |

### Custom Tasks
As part of the deployment, a few tasks are performed via custom code as they cannot be performed by the resource deployments themselves. Following you can find an overview of these tasks and the scripts you can leverage.

1. **Update WVD Host Pool**
   - **_Description:_** The script is designed to run more than once and will act differently depending on several parameters including deadline time and DeleteVM. The script first parses the host pool session hosts and matching VMs in Azure to get the value of the 'imageversion' tag. If the tag is absent or the value does not match the target version, then the session host will be evaluated. If the deadline is in the future, then each evaluated session host will be shutdown if there are no user sessions and if there are user sessions, then a logoff message is sent to each user with the deadline time. If the deadline has passed, then each evaluated session host will be shutdown if there are no user sessions and if there are user sessions, then each user will be forcefully logged off and the session host will be shutdown. If the 'DeleteVM' switch parameter is specified then the VMs (and associated NICs, OSDisks, and diagnostics storage accounts) will be deleted based on the deadline as above.
   - **_Script:_** `Update-WVDHostPool.ps1`