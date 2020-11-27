[[_TOC_]]

# Preparation
To ensure a successful deployment, we have one final step to perform: the deployment parameter configuration. Similar to the 'module pipelines', we have one parameters.json file per deployed service in addition to diverse pipeline variable files. 

## Generate parameter files

### Deployment parameters (.json files)

The following table lists the relevant parameters required as input to successfully deploy the _Key Vault_, _Storage Account_ and _WVD Host Pool_ resources:
<table>
    <thead>
        <tr>
            <th><b>File</b></th>
            <th><b>Location</b></th>
            <th><b>Recommended fields to change</b></th>
            <th><b>Description</b></th>
        </tr>
    </thead>
    <tbody>
        <tr valign="top">
            <td><code>keyvault.parameters.json</code></td>
            <td><code>Implementation/Environments/pipeline-orchestrated/wvdShared/parameters</code></td>
            <td>
               <ol style="list-style-type:circle;">
                   <li><code>keyVaultName</code></li>
  	      </ol>
            </td>
            <td>Hosts all parameters relevant for the Key Vault deployment.
            <br>A description of the default parameters can be found at</br>
		<ol style="list-style-type:disc;">
                    <li>ARM template reference <a href="https://docs.microsoft.com/en-us/azure/templates/microsoft.keyvault/2018-02-14/vaults">website</a></li>
                    <li>The parameters section of the ARM template at <code>'Modules/ARM/KeyVault/2019-11-28/deploy.json'</code></li>
		</ol>
            </td>
         </tr>
         <tr valign="top">
            <td><code>storageaccount.parameters.json</code></td>
            <td><code>Implementation/Environments/pipeline-orchestrated/wvdShared/parameters</code></td>
            <td>
               <ol style="list-style-type:circle;">
                       <li><code>storageAccountName</code></li>
                       <li><code>directoryServiceOptions</code></li>
                       <li><code>fileShareName</code></li>
	        </ol>
            </td>
            <td>Hosts all parameters relevant for the Storage Account deployment.
            <br>A description of the default parameters can be found at</br>
		<ol style="list-style-type:disc;">
		    <li>The parameters section of the ARM template reference <a href="https://docs.microsoft.com/en-us/azure/templates/microsoft.storage/2019-06-01/storageaccounts">website</a></li>
                    <li>The ARM template at <code>'Modules/ARM/StorageAccounts/2019-04-01/deploy.json'</code></li>
		</ol>
             The default SKU size is [Standard], but based on the workload [Premium] should be considered.
            </td>
        </tr>
        <tr valign="top">
            <td><code>wvd.parameters.json</code></td>
            <td><code>Implementation/Environments/pipeline-orchestrated/wvdShared/parameters</code></td>
            <td>
               <ol style="list-style-type:circle;">
                       <li><code>rdshNamePrefix</code></li>
                       <li><code>rdshNumberOfInstances</code></li>
                       <li><code>domainToJoin</code></li>
                       <li><code>existingDomainUPN</code></li>
                       <li><code>existingDomainPassword</code></li>
                       <li><code>existingVnetName</code></li>
                       <li><code>existingSubnetName</code></li>
                       <li><code>virtualNetworkResourceGroupName</code></li>
                       <li><code>existingTenantName</code></li>
                       <li><code>hostPoolName</code></li>
                       <li><code>enablePersistentDesktop</code></li>
                       <li><code>defaultDesktopUsers</code></li>
                       <li><code>tenantAdminUpnOrApplicationId</code></li>
                       <li><code>tenantAdminPassword</code></li>
                       <li><code>aadTenantId</code></li>
                       <li><code>fileShareStorageAccountName</code></li>
                       <li><code>fileShareStorageAccountNameRGName</code></li>
                       <li><code>windowsScriptExtensionFileUris<br>Token(s): [StorageAccountName]</br></code></li>
                       <li><code>windowsScriptExtensionCommandToExecute<br>Token(s): [StorageAccountName],[FileShareStorageAccountName],[domain],[targetGroup]</br></code></li>
	       </ol>
            </td>
            <td>Hosts all parameters relevant for the WVD Host Pool deloyment
            <br>A description of the default parameters can be found at</br>
		<ol style="list-style-type:disc;">
                    <li>The parameters section of the ARM template at <code>'Modules/ARM/WVD/2019-12-25/deploy.json'</code></li>
		</ol>
            </td>
        </tr>
    </tbody>
</table>




#### Azure ADDS vs native AD approach

Azure offers multiple storage solutions that you can use to store your FSLogix profile container. 
_This_ automation leverages Azure Files as a storage option for FSLogix.

Azure Files currently supports identity-based authentication over SMB protocol through two types of Domain Services: native Active Directory (AD) (preview) and Azure Active Directory Domain Services (Azure ADDS) (GA).

Based on the customer's identity and access management solution, refer to the following table for guidance on particular values to choose:


| Option | Parameter file| Location| Parameter name | Value|
| ------ | ------------- | ------- | -------------- | ---- |
| Native AD |  `storageaccount.parameters.json`| `Environments/pipeline-orchestrated/wvdShared/parameters` |  `directoryServiceOptions`  | `"None"`|
| Azure ADDS | `storageaccount.parameters.json`| `Environments/pipeline-orchestrated/wvdShared/parameters` |  `directoryServiceOptions` | `"AADDS"`|


When adopting the native AD approach, the above `directoryServiceOptions` value needs to be set to `"None"` in a first stage. Then, prior to the WVD host pool deployment, it is explicitly enabled following a semi-automatic approach, as detailed in the following sections.

>**Note**: this is not required in case FSLogix is not adopted.

#### FSLogix enabled vs. FSLogix disabled

Independently of the identity management in place (native AD or Azure ADDS) and the choice of host persistency (pooled or personal desktop), you may decide whether to deploy FSLogix or not. 

By design, the current version of this automation sets up an Azure files share to store user profiles and installs and configures FSLogix on the WVD host pool VMs. 

Refer to the following table for guidance on the relevant parameters and values to update:

| Option | Parameter file| Location| Parameter name | Value|
| ------ | ------------- | ------- | -------------- | ---- |
| FSLogix enabled | `'wvd.parameters.json'`| `Environments/pipeline-orchestrated/wvdShared/parameters` |  `fileShareStorageAccountName` <br/> `fileShareStorageAccountNameRGName` <br /> `windowsScriptExtensionFileUris` <br /> `windowsScriptExtensionFileUrisSasToken` <br /> `windowsScriptExtensionCommandToExecute`| Leave default and replace tokens `[wvdStorageAccount],[wvdStorageAccountResourceGroupName],[fileShareName],[domainToJoin],[fileshareUsersGroupName]`| 
| FSLogix disabled | `'wvd.parameters.json'`| `Environments/pipeline-orchestrated/wvdShared/parameters` |  `fileShareStorageAccountName` <br /> `fileShareStorageAccountNameRGName` <br /> `windowsScriptExtensionFileUris` <br /> `windowsScriptExtensionFileUrisSasToken` <br /> `windowsScriptExtensionCommandToExecute`| ""  <br/>  ""  <br/> []  <br/>  ""  <br/> ""| 



> **Warning**: In case you made any addition in the CustomScriptExention (CSE) section of the WVD ARM template, make sure that what explained above does not interfere with your customization.



#### Personal vs. Pooled Desktop

Based on the need to enable persistent desktop (personal desktop) or not (pooled desktop), refer to the following table for guidance on particular values to choose:

| Option | Parameter file| Location| Parameter name | Value|
| ------ | ------------- | ------- | -------------- | ---- |
| Personal | `'wvd.parameters.json'`| `'Environments/pipeline-orchestrated/wvdShared/parameters'` | `'enablePersistentDesktop'` | `true` |
| Pooled | `'wvd.parameters.json'` | `'Environments/pipeline-orchestrated/wvdShared/parameters'` | `'enablePersistentDesktop'` | `false` |



### Pipeline variables (.yml files)

The following table lists the relevant variables required as input to the pipeline to successfully deploy intended resources:


<table>
    <thead>
        <tr>
            <th>File</th>
            <th>Location</th>
            <th>Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><code>variables.shared.yml</code></td>
            <td><code>Environments/_Shared-Variables</code></td>
            <td>This file contains pipeline-related content that is relevant for all deployments. At least three values must be configured:
				<ol style="list-style-type:disc;">
				  <li><code>serviceConnection</code>
					<br>
					  DevOps Connection
					</br>
				  </li>
				  <li><code>componentStorageAccountSubscriptionId</code>
					<br>
					  Subscription ID of the storage account hosting the ARM templates used for the deployment
					</br>
				  </li>
				  <li><code>componentStorageAccountName</code>
					<br>Name of the storage account hosting the ARM templates</br>
				  </li>
				</ol>
            </td>
        </tr>
        <tr>
            <td><code>variables.shared.modules.yml</code></td>
            <td><code>Environments/pipeline-orchestrated/wvdShared/parameters</code></td>
            <td>This file contains pre- and post-deployment relevant content and is based on the previously configured module parameter files.
            </td>
        </tr>
    </tbody>
</table>

**Note**: Make sure that you use the same value for the same parameters across all files. This also includes the parameters used during the RDS Tenant deployment like the tenant name.

___

## (Optional) Automatically generate parameter files with PowerShell

To make your life a little easier, we provide a function as part of the WVDPreReq module that generates a base-version of the required parameter files for you. This is a two step process.
1. Define the desired parameters in the data file `'appliedParameters.psd1'` found in the `'.\static\'` folder 
2. Import the WVDPreReq module (`Import-Module -FullyQualifiedName '.\Implementation\WVDPreReq\WVDPreReq.psd1'`)
3. Invoke `'New-WvdPreReqPipelineParameterSetup -targetFolderPath '<desiredPath>'` with a folder of your choice to store the resulting files in. 

![generation.png](/.attachments/generation-833ffb52-63c6-4ef9-851e-4fdaa9ffa847.png =90%x)

The function consumes all files found in `'.\static\templates\parameters'`, replaces all enclosed tokens with the ones found in the `'appliedParameters.psd1'` file and exports it to the location specified as a parameter. Once finished, copy the parameter files over to the parameters folder of the deployment pipeline (e.g. `'Environments\pipeline-orchestrated\wvdShared\parameters\'`).
  In case you want to generate multiple host pool files for parallel pipeline deployments, just need to run the function several times with adjusted parameters and the `'templateSeachPattern'` parameter set to `".wvd.*"` and extract the resulting `'wvd.parameters.json'` file each time.