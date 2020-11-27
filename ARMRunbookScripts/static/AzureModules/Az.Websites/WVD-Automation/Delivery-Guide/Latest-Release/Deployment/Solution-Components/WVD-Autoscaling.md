
To improve the maturity of the deployed WVD solution and enable the customer to cut costs, the offer provides an optional auto-scaling feature. Following you will find an overview of the involved components, the flow and available configuration options.

The solution is based on a version published and documented by the product group [here](https://docs.microsoft.com/en-us/azure/virtual-desktop/virtual-desktop-fall-2019/set-up-scaling-script).

[[_TOC_]]

# Components
The feature requires two major components:
| | Resource | Resource Group | Description |
|--|--|--|--|
| ![Logic Apps.svg](/.attachments/Logic%20Apps-3d84b8e8-1896-4c7a-97f9-019293b14aaf.svg =15x) | WVD Scaling Scheduler | WVD-HostPool-RG | A Logic-App-Workflow deployed for each Host-Pool to trigger the host pool scaling. It contains a HashTable of all the data required to scale a particular Host Pool (e.g. Name of the Host Pool, designated Peak period, etc.). |
| ![Icon-manage-322-Automation-Accounts.svg](/.attachments/Icon-manage-322-Automation-Accounts-18077721-1b49-42d4-b5bd-9b1a8f0becf7.svg =15x) | Automation Account | WVD-Mgmt-RG | Contains the scaling runbook and is triggered by the 'WVD Scaling Scheduler(s)'. Responsible for scaling any given host pool. |)

---
---

# Process

In essence, the logic encapsulated in the scaling runbook controls the uptime of all session hosts that are part of the individual host pool. In other words, instead of spinning machines up and destroying them, the logic starts and stops VMs of a given pool.

The flow is as follows:

1. The _'WVD Scaling Scheduler'_ runs in a recurring interval (e.g. 15 minutes) and triggers a _'Webhook'_ in the automation account
2. The Webhook is associated with the _'Scaling Runbook'_ that hosts the actual scaling logic. The Host Pool it scales solely depends on the input parameters it receives from the webhook and in turn the _'WVD Scaling Scheduler'_ 
3. Based on the input parameters it received, the _'Scaling Runbook'_ starts to perform the appropriate actions on the _'Session Hosts'_ in the HostPool's resource group.

   ![scalingProcess.png](/.attachments/scalingProcess-dc7d9005-2736-4467-b6d0-b3d880d41f09.png =400x)

Once triggered, the runbook extracts all required input parameters from its WebHook and tries to calculate the current demand for active Session Hosts. In this context, the Peak vs. Off-Peak hours have the biggest influence on the scripts behavior.

As illustrated by following image, the runbook Scales-Out (aka starts Session Hosts) during Peak-Hours and Scales-In during Off-Peak hours. In an example with a 10 Session Host strong HostPool and a Peak-Periode from 9am to 6pm this means:
-  If the runbook is triggered at 8am it checks if the amount of active machines meets the configured minimum amount of always active machines. At 8am this should usually be the case, so nothing happens
- If the runbook is triggered at 10am, it checks again if the amount of active machines meets the configured minimum amount of always active machines + it checks if the current active user sessions meet the demand based on configured the amount of users per CPU per Session Host. If there are too few machines, the scripts starts to start new machines based on the demand. **NOTE:** The script does not deallocate machines during Peak hours.
- If the runbook is triggered again during the Off-Peak hours at e.g. 7 pm, it now tries to shut machines off. That means, for all machines that exceed the minimum configured amount of active hosts, it sends a message their current users to warn them that they will be logged off after a configured timeout. Once this deadline is reached, the script logs these users off and deallocates the hosts. 

  ![scalingClock.png](/.attachments/scalingClock-c3e8e3e3-ed88-4f6b-872d-10f81a175939.png =300x)

---
---

# Configuration

Following you can find an overview of all available configuration parameters for the _'Scaling Scheduler'_ webhook data object alongside a description of each:

| **Parameter Name** | **Type** | **Default Value** | **Mode** | **Example** | **Description** |
|--|--|--|--|--|--|
| `AutomationAccountName` | string | - | Required | `wvd-scaling-autoaccount` | Name of the automation account that hosts the scaling runbook | 
| `HostPoolName` | string | - | Required | `scalingHostPool` | Name of the HostPool to scale (should be in the same resource group as the _'Scaling Scheduler'_ workflow  | 
| `TimeDifference` | int | - | Required | `0` | The UTCOffset | 
| `BeginPeakTime` | string | - | Required | `9:00` | The designated start for the PeakTime period | 
| `EndPeakTime` | string | - | Required | `18:00` | The designated end for the PeakTime period  | 
| `LimitSecondsToForceLogOffUser` | int | - | Required | `300` | The time in seconds the user gets until the scaling script ends the user's session | 
| `LogOffMessageTitle` | string | - | Required | `Beware` | The title of the pop up the users receives when receiving the log-off warning | 
| `LogOffMessageBody` | string | - | Required | `Master Chief logs you off` | The message of the pop up the users receives when receiving the log-off warning | 
| `SessionThresholdPerCPU` | int | - | Required | `1` | The designated amount of users per CPU. Used to calculate to necessary amount of machines running | 
| `MinimumNumberOfRDSH` | int | - | Required | `1` | The minimum amount of Session Hosts to keep running at all times | 
| `MaintenanceTagName` | string | - | Optional | `IgnoreMe___Tag` | Tag to identify Session Hosts to ignore. | 
| `ResourceGroupName` | string | Resource Deployment ResourceGroup name | Optional. | `HostPool-01-RG` | Name of the HostPool resource group. | 
| `ConnectionAssetName` | string | `AzureRunAsConnection` | Optional | `AzureRunAsConnection` | Name of the automation account 'RunAs' connection. | 
| `AADTenantId` | Guid | Resource Deployment Tenant ID | Optional | `49ti757b-154a-464b-a576-e3f967e64a2a` | TenantId of the HostPool | 
| `subscriptionid` | Guid | Resource Deployment Subscription ID | Optional | `477b9620-cb01-114f-9ehf-fc6b1df48c42` | SubscriptionId of the HostPool | 
