<h1><span style="color:red">!!! UNDER CONSTRUCTION (SPRING UPDATE) !!!</span></h1>

# For the published version, please refer to the released version in the [master branch](https://dev.azure.com/SecInfra/Components/_wiki/wikis/WVD%20Automation/721/Welcome)

---

# WVD Automation – Delivery Guide

![intro.png](/.attachments/intro-fd9f5a27-f289-4c4b-9bf8-c510fc916db7.png =70%x)

This automated solution supports the following WVD capabilities:
- Full Desktop and RemoteApp
- Pooled / Personal
  - Parameter setup configures deployment
  - FSLogix still recommended for Personal desktops (e.g. for easier profile storage backup; VM re-assignment for immutable scenarios)
- Profile management (FSLogix)
  - FSLogix on Azure Files is part of the default deployment
  - Can be disabled
- Azure ADDS and Native Active Directory (ADDS)
  - Azure ADDS
    - GA for FSLogix on Azure Files
  - Native Active Directory (ADDS)
    - In Public Preview for FSLogix on Azure Files
- Any number of Host Pools of any size (same of different Subscription)
- Option to use multiple Storage Accounts for multi-host pool scenario
- Automation approach follows Services IaC principals

Typical deployment scenarios:

  | # | Identity solution | Configuration |
  |--|--|--|
  | 1 | Azure ADDS | Personal/Pooled desktops; <br/>With or without FSLogix |
  | 2 | Native AD | Personal desktops; <br>Without FSLogix |
  | 3 | Native AD | Personal/Pooled desktops; <br>With FSLogix|

# Areas covered in this Wiki
- [Delivery Guide](Delivery-Guide.md)
  - [Latest Release](/Delivery-Guide/Latest-Release)
    - [Deployment](/Delivery-Guide/Latest-Release/Deployment)
    - [Repository Structure & Flow](/Delivery-Guide/Latest-Release/Repository-Structure-&-Flow)
  - [2019 GA Release](/Delivery-Guide/2019-GA-Release)
    - [Deployment](/Delivery-Guide/2019-GA-Release/Deployment)
    - [Repository Structure & Flow](/Delivery-Guide/2019-GA-Release/Repository-Structure-&-Flow)
    - [Troubleshooting](/Delivery-Guide/2019-GA-Release/Troubleshooting)
    - [Examples](/Delivery-Guide/2019-GA-Release/Examples)

# Maintainers and Key Contributors
These are the Maintainers for this IP Component:
- Owner: @<13E518D2-C0BA-4352-A651-489CC093510F> 
- Technical Lead: @<3955223E-AA1F-4DC2-9294-22B63A26686E> 
- IaC Implementation: @<3955223E-AA1F-4DC2-9294-22B63A26686E>, @<603DE783-D07C-6BF7-8416-0B68F35E94EB>, @<1182D434-EB96-6C88-A0A5-4AB3C695704E> and @<E63791D6-B74C-6150-886A-4588ACA748C8> 

# IP Component Release Notes

## [V2006](https://dev.azure.com/SecInfra/Components/_workitems/edit/15048)
- #15240
- #15239
- #15242
- #15236

## [V2003](https://dev.azure.com/SecInfra/Components/_workitems/edit/14072)
### WVD PreRequisites
- #14852
- #14715
- #14714
- #14501
- #14279
- #14278
- #14275
- #14164

### WVD Structure
- #14277

### FSLogix Enablement
- #14280 

### Azure File with AD-Preview
- #14276