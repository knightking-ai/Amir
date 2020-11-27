The _'General Management'_ describes all items that are shared among several other features.

| | Resource | Resource Group | Description |
|--|--|--|--|
|  ![Icon-security-245-Key-Vaults.svg](/.attachments/Icon-security-245-Key-Vaults-4b958976-937a-405c-b158-1589745801e5.svg =15x)    | Key Vault | WVD-Mgmt-RG | Used by two other services: First, the Host-Pool deployment(s) to gather secrets like the default local admin password from. Second, the scaling deployment (automation account + WVD scaling scheduler) to store and retrieve webhook data. |
| ![Icon-storage-86-Storage-Accounts.svg](/.attachments/Icon-storage-86-Storage-Accounts-f10fc302-8c01-41ce-98c1-9f0a0e1c5389.svg =15x) | Storage Account | WVD-Mgmt-RG | Used to store assets used during the host pool deployment(s), like CSE scripts, and the scaling script used for the scaling runbook of the corresponding automation account. |
