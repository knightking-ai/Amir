This section is intended to give an overview of possible steps to perform once the deployment is finished.

[[_TOC_]]

# Remote Apps
The custom WVD schema in the WVDPreReq module describes four different levels of content:

::: mermaid
graph TD;
Tenant --> HostPool1
Tenant --> someHostPool["..."]
Tenant --> HostPooln["n-th HostPool"]
HostPool1 --> AppGroup1
HostPool1 --> someAppGroup["..."]
HostPool1 --> AppGroupn["n-th AppGroup"]
AppGroup1 --> RemoteApp1
AppGroup1 --> someRemoteApp["..."]
AppGroup1 --> RemoteAppn["n-th RemoteApp"]

   %% Style
   classDef tenant fill:#fbe6a6;
   classDef hostPool fill:#ceffbe;
   classDef appGroup fill:#6bf4ff;
   classDef remoteApp fill:#8eb3e7;

   class Tenant tenant;
   class HostPool1,someHostPool,HostPooln hostPool;
   class AppGroup1,someAppGroup,AppGroupn appGroup;
   class RemoteApp1,someRemoteApp,RemoteAppn remoteApp;
:::

As stated [before](https://dev.azure.com/SecInfra/Components/_wiki/wikis/Start%20Here/646/WVD-Service-Enablement?anchor=set-tenantstructure), we are easily able to add new HostPools and AppGroups to our tenant by modifying the `'appliedTenantStructure.json'` file in the `'/static/'` folder of the module and re-running `'Set-WVDPreReqTenantStructure'`. However, the logic only adds or updates content and does not remove elements form the tenant.
Something we can do now and have not been able before is to deploy `'RemoteApps'` to the tenant. 
For example:
```Json
 (...)
 "someAppGroup": [
    {
	"AppGroupName": "Office Application Group",
	"Description": "This is the default office App group",
	"FriendlyName": "Office-App-Group",
	"RemoteApp": [
            {
		   "RemoteAppName": "Excel",
		   "AppAlias": "excel"
            }
	]
     }
 ]
```

# User Assignment
Another important step of the WVD deployment is the eventual user assignment. This involves two different actions:

##  Add users to the group `'FileShareAccessGroup'`

> Required only if FSLogix is used

To have the user profiles synced successfully, the used SessionHost must be configured, and the user must be part of the group `'FileShareAccessGroup'`. The first condition is met by the previous deployment. The second can be met by querying through all intended users and add each to the `'FileShareAccessGroup'`via e.g. the PowerShell command `'Add-AzADGroupMember'`.

## Add users to the 'AppGroups' or 'SessionHosts'
The way of assigning users to tenant resources depends highly on the intended outcome. That is, whether to assign them to RemoteApp-AppGroups or DesktopApp-AppGroups.

### Desktop Application Group
The 'Desktop Application Group' is an auto-generated AppGroup used when you want users to have the Desktop experience when they log onto a machine. All you have to do, is to assign a user via the following command:

```PowerShell
$userAppGroupIntputObject = @{
   TenantName        = '<TenantName>'
   HostPoolName      = '<hostpoolname>'
   AppGroupName      = 'Desktop Application Group'
   UserPrincipalName = '<principalName>@<domain>.onmicrosoft.com'
}
Add-RdsAppGroupUser @userAppGroupIntputObject
```

#### Special Case: [Personal Desktop](https://docs.microsoft.com/en-us/azure/virtual-desktop/configure-host-pool-personal-desktop-assignment-type)
If you deployed the host pool with the `'AssignmentType'` : `'Direct'`, you have to go one step further and also assign this user to a specific session host. Once set, you are able to see the users as the assigned user of this host. The required command is:

```PowerShell
$sessionHostIntputObject = @{
   TenantName        = '<TenantName>'
   HostPoolName      = '<hostpoolname>' 
   Name              = '<sessionhostname>'
   AssignedUser      = '<principalName>@<domain>.onmicrosoft.com'
}
Set-RdsSessionHost @sessionHostIntputObject 
```

In contrast, if the `'AssignmentType'` is set to `'Automatic'`, a non-assigned machine is picked from the pool and WVD will automatically do the assignment for you.

### Other (Remote) Application Group(s)
If you want to provide users access to a pre-created app group with e.g. office apps, you just need to assign users to this app group. The command can look like:

```PowerShell
$userAppGroupIntputObject = @{
   TenantName        = '<TenantName>'
   HostPoolName      = '<hostpoolname>' 
   AppGroupName      = '<AppGroupName>'
   UserPrincipalName = '<principalName>@<domain>.onmicrosoft.com'
}
Add-RdsAppGroupUser @userAppGroupIntputObject
```
