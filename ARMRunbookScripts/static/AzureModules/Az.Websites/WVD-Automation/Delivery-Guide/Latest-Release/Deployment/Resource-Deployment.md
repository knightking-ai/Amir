With the environmental requirements completed, we can now continue with the resource deployment.

[[_TOC_]]

---

# Folder structure

Our suggested folder structure is as follows:
- On a top level you have an _'Environments'_ folder to group the individual deployments
- One level below you create a folder named after each resource group you want to deploy. E.g.
  - WVD-Imaging-RG
  - WVD-Mgmt-RG
  - WVD-Profiles-RG
  - WVD-HostPool-01-RG
- Each resource group folder should, in turn, contain all relevant artifacts (e.g. parameter files, pipeline files, etc.) to deploy the resources of this resource group.

---
---

# Modes

As part of this solution we offer two different deployment modes:

- _**pipeline-orchestrated**_: The deployment of resources using ARM is split into individual pipeline jobs and steps. The order in which resources are deployed and what scripts are executed is fully controlled by the pipeline setup.

- _**template-orchestrated**_: The deployment of resources using ARM is orchestrated by 'master' ARM templates by leveraging linked-templates. The order in which resources are deployed is fully controlled by these templates. DevOps pipelines are used to trigger the template deployment.

Depending on the approach you choose, the deployment setup differs and for example parameters have to be applied in different locations. The following sections will go into greater detail for each approach.

## Template-Orchestration

Each resource group folder should have the following files:

| File Name | Description |
|--|--|
| `pipeline.yml` | _Required_. A baseline DevOps YAML pipeline used to validate and deploy the master ARM template. |
| `variables.yml` | _Required_. A variables file used by the pipeline containing pipeline relevant data like the used service connection and agent. |
| `deploy.json` | _Required_. The master template that performs and controls resource deployment of each resource that is part of this resource group. |
| `parameters.json` | _Optional_. A parameter file used for additional parameters like key vault secret references. |

![TemplateOrchestration Overview.png](/.attachments/TemplateOrchestration%20Overview-80566651-ca16-46fa-b637-1ece2a859be3.png =60%x)

---

## Pipeline Orchestration

Each resource group folder should have the following files

| File Name | Description |
|--|--|
| `pipeline.yml` | _Required_. This pipeline file references the module templates and orchestrates the deployment of all resources contained in the resource group. |
| `variables.yml` | _Required_.  A variables file used by the pipeline containing pipeline relevant data like the used service connection and agent, as well as variables used by any used script |
| `Parameters/<resource>.parameters.json` | _Required_. Each resource deployment in the `pipeline.yml` references a parameter file in the 'Parameters' folder. |
| `Scripts/Invoke-<resource><Post/Pre>Deployment.ps1` | _Optional_. As part of the deployment it may be necessary to run scripts (e.g. as a post-deployment to a key vault deployment). Hence the `pipeline.yml` references scripts in the 'Scripts' folder. |

![PipelineOrchestration Overview.png](/.attachments/PipelineOrchestration%20Overview-c6a0d304-dfb1-445f-800a-cb5a8ec3d91e.png =60%x)

The pipeline-orchestration relies on three pillars:
1. Order and orchestration of deployments via the `pipeline.yml`
2. The `parameter.json` files that are used during the ARM deployments
3. The `variable.json` that holds information to feed into the pipeline, but also provides variables to any scripts we want to run there (e.g. Post-Deployment scripts).


### How-To: Add a new resource-deployment to the pipeline
If you want to add a new resource-deployment you need two things: add a new parameter file and reference it in the pipeline.

The parameter files are just the classic ARM template parameter files and you should add the file you want to use to the 'Parameters' folder of the ResourceGroup folder where you want to add the resource-deployment. 

Regarding the pipeline, each new resource-deployment should be added as a new job/deployment to to the pipeline to make use of the job-parallelization if available. The only thing to keep in mind when doing this is to properly configure any dependencies in between jobs where necessary (e.g. a VM deployment should follow its VNET deployment).

Without any other caviars, the YAML code you want to add looks like this:

```YAML
  - deployment: Deploy_<<<MeaningfulResourceName>>>
    dependsOn: ''
    environment: <<<Environment>>>
    condition: and(succeeded(), true) 
    timeoutInMinutes: 120
    pool:
      vmImage: $(vmImage)
    strategy:
        runOnce:
          deploy:
            steps:
              - checkout: self
              - task: AzurePowerShell@4
                displayName: 'Deploy module [<ResourceName>] in [<<<ResourceGroupVariableName>>>] via [$(serviceConnection)]'
                name: Deploy_<MeaningfulResourceName>_Task
                inputs:
                  azureSubscription: $(serviceConnection)
                  ScriptType: InlineScript
                  inline: |
                    Write-Verbose "Load function" -Verbose
                    . '$(Build.Repository.LocalPath)/$(orchestrationFunctionsPath)/Invoke-GeneralDeployment.ps1'

                    $parameterFilePath = Join-Path '$(Build.Repository.LocalPath)' '$(rgFolderPath)/Parameters/wvdapplication.parameters.json'
                    $functionInput = @{
                      resourcegroupName             = "<<<ResourceGroupVariableName>>>"
                      location                      = "<<<LocationVariableName>>>"
                      componentStorageAccountName   = "$(componentStorageAccountName)"
                      componentStorageContainerName = "$(componentStorageContainerName)"
                      moduleName                    = "<<<ResourceName>>>" # In template storage account
                      moduleVersion                 = "<<<ResourceVersion>>>" # In template storage account
                      parameterFilePath             = $parameterFilePath
                    }

                    Write-Verbose "Invoke task with" -Verbose
                    $functionInput.Keys | ForEach-Object { Write-Verbose ("PARAMETER: `t'{0}' with value '{1}'" -f $_, $functionInput[$_]) -Verbose }

                    Invoke-GeneralDeployment @functionInput -Verbose
                  errorActionPreference: stop
                  azurePowerShellVersion: LatestVersion
                enabled: true
``` 

The important keywords to point out are:
- **MeaningfulResourceName:**: A meaningful name for the dpeloyment that will be visible in the UI (e.g. WVDKeyVault)
- **Environment**: The environment this deployment points to. Enables you to define approvers (e.g. 'SBX')
- **ResourceGroupVariableName:** The name of the resource group to deploy to. Should be a variable in the variable file and be referenced via `'$(resourceGroupName)'`
- **LocationVariableName:** Location to deploy to. Should be a variable in the variable file and be referenced via `'$(location)'`
- **ResourceName:** The name of the resource to deploy. Must match the name it has in the template storage account to enable its deploy.json reference (e.g. KeyVault).
- **ResourceVersion:** The version of the resource to deploy. Must match the version it has in the template storage account to enable its deploy.json reference (e.g. 1.0.0).

### How-To: Add a new script to the deployment
Just as for any other YAML pipeline you can add new tasks any place you want. If you only want to add an independent script you can create a new job with only a PowerShell task inside. In contrast, if you want the script to run as a post-deployment to an existing resource-deployment, it is easier to just add the task right after the `'Deploy_<MeaningfulResourceName>_Task'` step.

By default, pipeline tasks can use of variable in the pipelines `variables.yml` file. So if you need any input for you script, the easiest way to make it available is to get it from this file. However, if you need deployment outputs from other steps in the pipeline, you can also use the [variable handover](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#share-variables-across-pipelines) provided by DevOps 

### How-To: Deploy resources to different locations within one pipeline

The location for the resource-deployments is decided by the `'FunctionInput'` property 'Location' like in the example ResourceDeployment above:
```PowerShell
$functionInput = @{
  (...)
  location = "<<<LocationVariableName>>>"
  (...)
}
Invoke-GeneralDeployment @functionInput -Verbose
```

As stated before, it is recommended to reference a variable from the pipelines 'variable.yml' here and reference it via `'$(<<<variableName>>>)'`. 

That said, the only action you have to perform to add an additional location is to add another variable to the pipelines 'variable.yml' file and reference is with the above syntax.

For example
- Definition in the pipelines 'variable.yml' file
  ```Yaml
  defaultLocation  = 'WestEurope'
  hostPoolLocation = 'EastUs2'
   ```

- Reference in the 'pipeline.yml' file
  ```PowerShell
  location = '$(defaultLocation)'
  location = '$(hostPoolLocation)'
  ```

### How-To: Pass variables into parameter-based deployments
#### Parameter Value
In order to pass full parameter values (e.g. an image reference for the Session Host deployment) into a deployment that is performed by the `'Invoke-GeneralDeployment.ps1'` script, you can make use of the `'optionalParameters'` function parameter. This parameter must **NOT** be also specified in the used `parameter.json`

Example:
```PowerShell
$functionInput  = @{
   # (...)
}

$optionalParameters = $()
if (-not $someCondition) {
  $optionalParameters += @{ imageReference = @{ id = '$(customImageReferenceId)' } }
}
else {
  $imageReference = @{
	publisher = '$(publisher)'
	offer     = '$(offer)'
	sku       = '$(sku)'
	version   = '$(version)'
  }
  $optionalParameters+= @{ imageReference = $imageReference }
}
$functionInput += @{
  optionalParameters = $optionalParameters
}
```

#### Parameter-Object Property

In some instances, it might be necessary to not replace an entire object, but only a property of a provided parameter object. To enable this scenario, we added the function `Add-CustomParameters.ps1`. In contrast to the previous ['Parameter Value'](#Parameter-Object-Property) approach this function makes the values available by replacing the intended values inside the `parameters.json` file and stores it on disk.

```PowerShell
<# DummyFile
{
   (...)
   "parameters": {
       "dscConfiguration": {
          "value": {
              "protectedSettings": "I will be replaced"
          }
       },
       "somePath": {
          "value": "The token [sasKey] gets replaced"
       }
   }
}
#>

# Example 1: Replacing the value in the parameters path '[dscConfiguration.value.protectedSettings.configurationArguments.registrationInfoToken]'
# Here: Assumes you have a parameter 'parameter.dscConfiguration.value.protectedSettings' and want to give it the value 'someProtectedSetting'
$overwriteInputObject = @{
  parameterFilePath = $parameterFilePath
  valueMap         = @(
	@{ 
           path  = 'dscConfiguration.value.protectedSettings';  
           value = 'someProtectedSetting'
        }
  )
}
Add-CustomParameters @overwriteInputObject

# Example 2: Replace a token inside a given property string
# Here: Assumes you have a parameter 'parameters.somePath' with a token '[sasKey]' (e.g. "The token [sasKey] gets replaced") and want so replace the token with the value 'someValue'
$overwriteInputObject = @{
  parameterFilePath = $parameterFilePath
  valueMap         = @(
	@{ 
           path         = 'somePath'
           value        = 'someValue'
           replaceToken = '[sasKey]'
        }
  )
}
Add-CustomParameters @overwriteInputObject

<# Dummy result file after function execution
{
   (...)
   "parameters": {
       "dscConfiguration": {
          "value": {
              "protectedSettings": "someProtectedSetting"
          }
       },
       "somePath": {
          "value": "The token someValue gets replaced"
       }
   }
}
#>
```