This section describes what is required to deploy and manage the WVD environment using Azure DevOps.

> **NOTE:**
> The configuration of these steps is a **shared responsibility**: Azure DevOps project has to be prepared by the customer; MCS supports code onboarding and related steps.

[[_TOC_]]

---

<span style="color:red">[<TODO>: ADD DevOps IMAGE]</span>

# Azure DevOps prerequisites

> ![svg-azure-devops.svg](/.attachments/svg-azure-devops-51b9c072-f03f-48b6-aa18-6dd235edc97b.svg =15x)

The prerequisites in this section have to be prepared by the customer:
- Dedicated DevOps Project / Sufficient rights on existing project (e.g. ACF): the customer has to have an Azure DevOps project prepared to onboard the WVD DevOps automation solution.
- User Licences for DevOps: each person who has to have access to Azure DevOps to write and maintain code or deploy services to Azure using pipelines, has to have a license assigned. The customer has to have these in place to allow the delivery team to work.
- Pipeline Agent availability: to run the deployment pipelines that are part of this solution, the customer has to have an available pipeline agent at the time of the deployment. This agent can be Microsoft-hosted or even self-hosted, as long as it has all required components available (e.g. Git, PowerShell with AZ module).
- Service Connection or _'Deployment Service Principal'_ :
  To enable pipelines to deploy resources to an Azure subscription we need a service connection. The AAD Service Principal used can be created in the AAD tenant prior to the start of the project or together with the customer. However, to create the service connection based on a Service Principal that was set up before, you need to get hold of its Application ID (Client ID) and secret (or create one if not existing).
  - Required Permissions

    | Scope| Configuration | Role |
    |--|--|--|
    | Subscription | RBAC (IAM) | - User Access Administrator <br/> - Contributor|
    | AzureAD | Roles | - Application Developer <br/> - User administrator |
    | AzureAD  | API Permissions | Azure Active Directory Graph: <br/> - Application: Application.ReadWrite.OwnedBy |
    | AzureAD  | API Permissions | Microsoft Graph: <br/> - Application: Application.ReadWrite.OwnedBy <br/> - Delegated:  Group.ReadWrite.All <br/> - Delegated: User.Read <br/> - Delegated: User.ReadWrite |

- Tooling requirements: installing and using Git and VS Code with access to the Extension gallery and the latest PowerShell modules must be allowed. These tools have to have access to the internet.

---
---

# Code onboarding
- [Create a new '_Components_' repository](#Create-a-new-'_Components_'-repository)
- [Push code to the customer's '_Components_' repository](#Push-code-to-the-customer's-'_Components_'-repository)
- [Upload 'Modules' folder to the '_Components_' Storage Account](#Upload-'Modules'-folder-to-the-'_Components_'-Storage-Account)
- [Set up the release pipeline](#Set-up-the-release-pipeline)
- [Native AD DevOps secret storing Domain Join Service Account Password](#Native-AD-DevOps-secret-storing-Domain-Join-Service-Account-Password)

## Create a new '_Components_' repository
> ![svg-azure-repos.svg](/.attachments/svg-azure-repos-17353519-d9be-451e-9397-dc6fd89e5af5.svg =15x)

In the customer's ADO project, create a new Git repository, called '_Components_'.

---

## Push code to the customer's '_Components_' repository
> ![svg-azure-repos.svg](/.attachments/svg-azure-repos-17353519-d9be-451e-9397-dc6fd89e5af5.svg =15x)

To upload the '_WVD DevOps automation_' code base into the 'Components' repository in the customer's Azure DevOps environment, follow these steps:
- Download the content of the [WVD DevOps automation code base](https://aka.ms/WVD-DevOps-Automation-Code) as a zip file.
- Clone the customer's 'Components' repository.
- Expand the zip file to the customer's cloned repo.
- Commit the changes and push the code to the customer's repo.

---

## Upload 'Modules' folder to the '_Components_' Storage Account

> ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x)

Following the Infrastructure-as-Code approach based on which ACF (Azure Cloud Foundation) projects are delivered, the pipeline used in this solution expects a storage account with a blob storage containing all ARM templates we want to deploy. That said, the suggested approach is to create this storage account in the customer's environment and upload the required tested ARM templates to it. 

### -_Option 1_- Manual upload
1. Make sure you create this storage account with a unique name in the target subscription.
2. Add a blob container called "Components" to it.
3. Upload the full '**Modules**' folder from the 'Components' repository.

### -_Option 2_- Set up the build pipeline for each module
> **Beware**: Though it is good to have a testing process in place, this setup can be time consuming to configure. If time is at the essence, start with the manual upload.

To have the module test pipelines ready we have to configure the build pipeline parameters. This action requires us to look into several places:

- Inside the 'Modules/ARM' folder you can find a couple variables in the `'Pipeline/pipleine.yml'` file. Some of the most important ones are:

   ```YAML
   - name: serviceConnection
     value: <ServiceConnection>

   - name: componentStorageAccountSubscriptionId
     value: <ID>

   - name: componentStorageAccountName
     value: <componentStorageAccountName>
   ```

  You need to set the name of your DevOps service connection, the subscription id of the subscription containing the 'Components' storage account [you created previously](#'_Components_'-Storage-Account) and the name of this storage account. 

- The setup works similar to [the previous section](#Set-up-the-release-pipeline), but instead of referencing a path in the 'Environments' directory, select the path to a module pipeline (e.g. `'Modules/ARM/WVD/2019-12-25/Pipeline/pipeline.yml'`). Also, add a pipeline variable `'RemoveDeployment'` with the value of 'true' and the 'Let users override this value when running this pipeline' flag set to true. This enables you to decide on deployment whether the deployed test resources should be removed or not when the pipeline is run. Perform this action for all modules you want to be able to test via a pipeline (e.g. WVD, KeyVault and StorageAccount). Also make sure, the correct 2 variable paths are set at the top of the pipeline file. The second one has to point towards the one configured in the previous step:
   ```YAML
   variables:
   - template: pipeline.variables.yml
   - template: ../../../variables.yml
   ```

- Next up, we have to configure the deployment parameters for the test deployments. In each module folder you can find a folder `'Parameters'` with a `'parameters_generic.json'` in it. As before, set all values enclosed by '<>'. This file must be referenced in the `'pipeline.yml'` as well. So make sure for each row with `parametersFile: $(modulePath)/Parameters/parameters.json` you reference the configured parameters file.

By default, all test pipelines are automatically executed whenever changes happen to their module path in branch master.

---
---

# Set up the deployment pipeline

> ![svg-azure-devops.svg](/.attachments/svg-azure-devops-51b9c072-f03f-48b6-aa18-6dd235edc97b.svg =15x)

With the 'Components' repository in place, you have access to the deployment pipeline that will deploy the WVD solution. 

To register this pipeline in Azure DevOps,  
   - Navigate to the '_Pipelines_' section of _Azure DevOps_ portal
   - Select '_New pipeline_'
   - Select  '_Azure Repos Git_'
   - Select the repository you uploaded the code to
   - Select '_Existing Azure Pipelines YAML file_'
   - Copy the folder path to the pipeline ('Environments' directory) into the 'Path' dropdown (e.g. `'Environments/pipeline-orchestrated/wvdShared/custom.pipeline.yml'`)
   - Select '_Continue_' and save the pipeline

---
---

# Storing secrets in DevOps _Variable Group_

![image.png](/.attachments/image-cbb2a506-a581-4cd0-8914-c2a48ff5519a.png =300x)

1. Create a variable group
      - Navigate to the '_Library_' page under the '_Pipelines_' section of _Azure DevOps_ portal
      - Select '_+ Variable Group_' to add a new variable group
      - Choose a name for the group (e.g. WVDSecrets) and a description
        > **Note**: Keep track of the name chosen for the variable group as this needs to be added in the pipeline variables as described in the [HostPool deployment](/Delivery-Guide/Latest-Release/Deployment/Resource-Deployment/Host%2DPool-Resource-Group) section.
2. Add a variable secret  
      - Select '_+ Add_' in the '_Variables_' section to add a new variable
      - Insert the required secret name below in the '_Name_' field. This exact name is required for the successful run of the pipeline
      - Insert the corresponding secret in the '_Value_' field.
      - Select the Lock on the right to change the variable type to secret and Save

## Secrets
> Note Please pay attention to the correct name required for this secret, as detailed below.

<table>
    <thead>
        <tr>
            <th><b>Secret </b></th>
            <th><b>Naming</b></th>
            <th><b>Description</b></th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>
<div style="padding: 10px; border-left: 1px solid #FFC000;">Native-AD Domain Join Service Account Password</div></td>
            <td>WVDJoinDomainADPwd</td>
            <td>This secret is required to securely store the <code>'Domain Join Service Account'</code> credentials in a DevOps secret. The secret is needed by the deployment to perform the domain join of the WVD host pool VMs.</td>
         </tr>
         <tr>
            <td><div style="padding: 10px; border-left: 1px solid #85DFFF;">Azure-ADDS Domain Join Service Account Password</div></td>
            <td>WVDJoinDomainADPwd</td>
            <td>This secret is required to securely store the <code>'Domain Join Service Account'</code> credentials in a DevOps secret. The secret is needed by the deployment to perform the domain join of the WVD host pool VMs.</td>
         </tr>
         <tr>
            <td>(optional) Default Local Admin Password</td>
            <td>localAdminPassword </td>
            <td>Using this variable you can provide a local admin password for the WVD session host VMs. If no value is provided, the Mgmt-RG deployment will generate a password for you and store it in this resource groups key vault.</td>
         </tr>
     </tbody>
</table>