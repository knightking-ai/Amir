[[_TOC_]]

# Operation Errors
## FSLogix User Profile not synchronized
### Possible causes
- **Current user profile was created on machine without being member of the file share user group.**
  - Mitigation: Remove current user profile (requires admin permissions) and log-off and -on again

- **File share was not correctly set-up on not even one host pool**
  - Mitigation: Please double-check 
    -  The log files at `'C:\Windows Azure\Custom Script Extension\*FileShare*'`. If everything went good, the drive permissions should be shown on the bottom and it should also mention `Successfully processed 1 files; Failed processing 0 files`.
    - The storage account is successfully set up and its correct values are specified in the WVD parameters json: `storageAccountResourceGroupName` & `storageAccountName`
    - The required files made int to the file share blob of the same storage account (should not be empty). Also, check if the correct path is specified in the WVD parameters json: `windowsScriptExtensionFileUris`
    - Make sure the `windowsScriptExtensionCommandToExecute` is correctly set. Especially regarding the group parameter. You can also double-check what values were used in the other logs at `'C:\Windows Azure\Custom Script Extension\*FileShare*'`
  
    If any of the above commands went wrong, fix the incorrect parts and try to re-run the custom script extension.