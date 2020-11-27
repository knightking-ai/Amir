The _'Host-Pool'_ and all related components are the main component of this solution. It takes care of enable the WVD solution for users.

| | Resource | Resource Group | Description |
|--|--|--|--|
| ![wvdHostPool.png](/.attachments/wvdHostPool-09c9de61-6232-4783-ae58-a53978ed2221.png =15x) | Host Pool | WVD-HostPool-RG | A central resource to link several WVD resources like the session hosts and application groups together. |
| ![wvdApplicationGroup.png](/.attachments/wvdApplicationGroup-7d566151-21b7-4e56-bc55-93d67fafa78c.png =15x) | Application Group | WVD-HostPool-RG | Can be both a Desktop-Application-Group for desktop users and Remote-Application-Group for application users. |
| ![wvdWorkspace.png](/.attachments/wvdWorkspace-cb10f06d-1f7e-4d36-8f4d-018158c76bda.png =15x) | Workspace | WVD-HostPool-RG | Application Groups linked to a workspace show up as published in the WVD client. |
| ![vm.png](/.attachments/vm-47c5cd9a-6e8a-4ac6-bbfd-8ebd34a5e2fa.png =15x) | Session Host | WVD-HostPool-RG | VMs that carry the WVD workloads (e.g. user sessions). |
| ![wvdApplication.png](/.attachments/wvdApplication-6d89712a-ff94-4633-8e32-fdb80b234b55.png =15x) | Application | WVD-HostPool-RG | Links applications that are installed on the session host(s) to a Remote-Application-Group. |
