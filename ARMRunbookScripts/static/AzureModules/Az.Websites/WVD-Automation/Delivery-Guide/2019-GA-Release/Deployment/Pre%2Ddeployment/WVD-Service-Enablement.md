This session describes the actions that are required by the customer's Global Administrator before any WVD deployments can take place - therefore these steps will be performed together with the customer.

[[_TOC_]]

![prereqwvd.png](/.attachments/prereqwvd-167ebffd-8dab-45ed-a9bb-f9d225dcf74d.png)

# WVD Enterprise Applications

> ![Icon-identity-232-App-Registrations.svg](/.attachments/Icon-identity-232-App-Registrations-0053063a-f865-4239-b962-e2d87a5cd581.svg =15x)

As [previously mentioned](https://dev.azure.com/SecInfra/Components/_wiki/wikis/Start%20Here/638/Prerequisites?anchor=azure-environment), WVD requires two applications to be registered in the tenant prior to any deployment. The _Global Administrator_ has to register both using the corresponding [website](https://rdweb.wvd.microsoft.com/):    
- _'Windows Virtual Desktop'_ Enterprise Application
- _'Windows Virtual Desktop Client'_ Enterprise Application

#_WVDPreReq_ Module

> ![svg-cloud-shell.svg](/.attachments/svg-cloud-shell-d1160562-1662-4804-9142-dfe3bffa4ca5.svg =15x)

> Note: The module has the default command prefix `'WVDPreReq'`, i.e. once imported, you must invoke all functions of the public folder (exported functions) with the prefix. E.g. 'Get-Name' becomes <code>'Get-<b>WVDPreReq</b>Name'</code>

> Note: The module is located at `'Implementation/WVDPreReq'`

Once imported, the _WVDPreReq_ module offers a number of functions. You will need to use the below three going forward: 
- [Install-RequiredModule](#Install-RequiredModule)
- [Set-Up](#Set-Up)
- [Set-TenantStructure](#Set-TenantStructure)
- [Test-TenantStructureConfig](#Test-TenantStructureConfig)

---

## Install-RequiredModule

  This function installs a given PowerShell module if not present. 

### Parameters

 | Parameter | Description |
 | --------- | ----------- |
 | _**ModuleName**_ | The name of the module to install if not already |

### Example

```PowerShell
> Install-WVDPreReqRequiredModule -ModuleName 'AzureAD'
```

---

## Set-Up 

This function sets up the remaining prerequisites for the WVD deployment, including:
  | Name                                                                                     | Description                                                             |
  |------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
  | Assign TenantCreator to `[Global Administrator]`                                         | A WVD _TenantCreator_ AppRole assignment for the current signed in user |
  | Create RDS Tenant                                                                        | Creation of the RDS tenant (=WVD tenant)                                |                        
  | Application: Tenant Admin Service Principal                                              | Creation of the WVD _Tenant Admin Service Principal_                    |
  | `[Tenant Admin Service Principal]` Role Assignment (RDS Owner)                           | Assignment of the RDSOwner role to this Service Principal               |
  | [Deployment Service Principal] as owner assignment to `[Tenant Admin Service Principal]` | Assignment of the provided deployment Service Principal as an owner to this Service Principal |

The execution requires:

- All required PowerShell modules to be present (`Az`, `AzureAD` & `Microsoft.Rdinfra.RdPowerShell`), 
- A Global Administrator to run the script
- A Global Administrator to be logged in and having at least 'Reader' access to target subscription. Background: During module execution we fetch the tenant id from the active context.

### Parameters

| Parameter | Description |
| --------- | ----------- |
| _**WvdTenantName**_ | The name of the RDS tenant to deploy |
| _**TenantAdmin_ServicePrincipalName**_ | The name of the Service Principal to act as an RDS owner / _Tenant Admin Service Principal_ |
| _**DeploymentServicePrincipalName**_ | The name of the deployment Service Principal used by the DevOps solution / Service Connection
  
### Example

```PowerShell
> Set-WVDPreReqUp -WvdTenantName "ContosoWVDTenant" -TenantAdmin_ServicePrincipalName "TenantAdminServicePrincipal" -DeploymentServicePrincipalName "wvdDeploymentServicePrincipal"
```
  
---

## Set-TenantStructure

This function can only be invoked after the `Set-WVDPreReqUp` script was run. 

If executed without any other parameter, it essentially takes the `'appliedTenantStructure.json'` file found in the `'.\static\'` folder and attempts to set up the tenant schema defined. This includes one or multiple tenants with one or multiple host pools and one or multiple app groups.  

That said, the structure must be defined prior to executing the function. Though several later changes are possible, the function only adds or updates tenant data, in the current implementation it does not remove any items.

The structure should follow the schema defined in the `'wvdTenant-schema.json'` file, also found in the `'.\static\'` folder. 

**IMPORTANT**: Make use of the "RemoteApp" object only AFTER you deployed the first couple session hosts (aka WVD VMs). Until then, they are not supported.

Make sure you are logged into the RDSTenant when using this script.
 
### Example

```PowerShell
> Set-WVDPreReqTenantStructure
```

### Personal Desktop vs. Pooled Desktop

Depending on the type of desktop type you need to deploy, you'll have to have configure certain parameters in the  `'appliedTenantStructure.json'` file before creating the tenant structure:

-  `Persistent`
   - Set to `'true'` for personal desktop
- `AssignmentType`
   Especially relevant for personal desktops. Decides how users are assigned to hosts.
  - If users should be assigned to session hosts by hand: `Direct` (for personal desktop this assignment is permanent)
  - If users should be assigned automatically to hosts: `Automatic` (for personal desktop this assignment is permanent)

### Execution as Service Principal
With the TenantAdmin-ServicePrincipal deployed and configured, the script can also be executed in this service principals context. The one piece of information you need is the service principals secret. In case the later KeyVault deployment was already executed, you can find the secret in this key vault. Otherwise you can create one manually. To log into the RDS-Tenant as a service principal, use the following script:
```PowerShell
$aadTenantId = '<AAdTenantId>'
$wvdSpnAppId = '<AppId>'
$wvdSpnPwd = '<SecretAsSecureString>'

$creds = New-Object System.Management.Automation.PSCredential($wvdSpnAppId, $wvdSpnPwd)
Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com" -Credential $creds -ServicePrincipal -AadTenantId $aadTenantId
```

---

## Test-TenantStructureConfig

When creating the structure file, you can use the public module function `'Test-TenantStructureConfig.ps1'` to validate it towards the schema. 

> **NOTE: This feature requires PowerShell Core !**

### Example

```PowerShell
> Test-WVDPreReqTenantStructureConfig
``` 