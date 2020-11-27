[[_TOC_]]

# Deployment Errors

## Join Domain Extension fails
### Example Output
```Json
    {
        "code": "ComponentStatus/JoinDomainException for Option 1 meaning 'User Specified without NetSetupAcctCreate'/failed/1",
        "level": "Error",
        "displayStatus": "Provisioning failed",
        "message": "ERROR - Failed to join domain='cedward.onmicrosoft.com', ou='', user='domainJoinUser@<domain>.onmicrosoft.com', option='NetSetupJoinDomain' (#1 meaning 'User Specified without NetSetupAcctCreate'). Error code 1326"
    }
```
### Possible causes
- **The same user is set up as `'existingDomainUPN'` & `'defaultDesktopUsers'`**
  - Mitigation: Use a different user for the `'defaultDesktopUsers'`
- **DNS Server not set to AADDS in VNET properties**
  - Mitigation: Set the `'DNS server'` entries on the VNET to the ones found in on the AADDS page at `'Properties\IP address on virtual network'`
- **WVD-Subnet has no AADDS endpoint enabled**
   - Go through the WVD subnet(s) and set the Endpoint to "Microsoft.AzureActiveDirectory"

## DSC Extension Fails - [Host Pool not found]
### Example Output
```Json
{
   "PowerShell DSC resource MSFT_ScriptResource failed to execute Set-TargetResource functionality with error message: User is not authorized to query the management service.\nActivityId: 40b1f9df-5279-40e3-9fe1-29565571df52"
   "PowerShell DSC resource MSFT_ScriptResource failed to execute Set-TargetResource functionality with error message: myHostPool Hostpool does not exist in <TenantName> Tenant  The SendConfigurationApply function did not succeed."
} 
```
### Possible causes
- **New tenant created without 'RDS Owner' role assignment to deployment principal**
  - Mitigation: Assign 'RDS Owner' role to principal via command: 
       `New-RdsRoleAssignment -RoleDefinitionName "RDS Owner" -ApplicationId "<AppId>" -TenantName "<TenantName>"` (Service Principal example)