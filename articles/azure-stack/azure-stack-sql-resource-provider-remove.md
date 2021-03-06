---
title: Removing the SQL resource provider on Azure Stack | Microsoft Docs
description: Learn how you to remove the SQL resource provider from your Azure Stack deployment.
services: azure-stack
documentationCenter: ''
author: jeffgilb
manager: femila
editor: ''

ms.service: azure-stack
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 11/14/2018
ms.author: jeffgilb
ms.reviewer: quying

---

# Remove the SQL resource provider

Before you remove the SQL resource provider, you must remove all the provider dependencies. You'll also need a copy of the deployment package that was used to install the resource provider.

  |Minimum Azure Stack version|SQL RP version|
  |-----|-----|
  |Version 1808 (1.1808.0.97)|[SQL RP version 1.1.30.0](https://aka.ms/azurestacksqlrp11300)|
  |Version 1804 (1.0.180513.1)|[SQL RP version 1.1.24.0](https://aka.ms/azurestacksqlrp11240)
  |     |     |

## Dependency cleanup

There are several cleanup tasks to do before you run the DeploySqlProvider.ps1 script to remove the resource provider.

Azure Stack tenant users are responsible for the following cleanup tasks:

* Delete all their databases from the resource provider. (Deleting the tenant databases doesn't delete the data.)
* Unregister from the provider namespace.

The Azure Stack Operator is responsible for the following cleanup tasks:

* Deletes the hosting servers from the MySQL Adapter.
* Deletes any plans that reference the MySQL Adapter.
* Deletes any quotas that are associated with the MySQL Adapter.

## To remove the SQL resource provider

1. Verify that you've removed all the existing SQL resource provider dependencies.

   > [!NOTE]
   > Uninstalling the SQL resource provider will proceed even if dependent resources are currently using the resource provider.
  
2. Get a copy of the SQL resource provider binary and then run the self-extractor to extract the contents to a temporary directory.

3. Open a new elevated PowerShell console window and change to the directory where you extracted the SQL resource provider binary files.

4. Run the DeploySqlProvider.ps1 script using the following parameters:

    * **Uninstall**. Removes the resource provider and all associated resources.
    * **PrivilegedEndpoint**. The IP address or DNS name of the privileged endpoint.
    * **AzureEnvironment**. The Azure environment used for deploying Azure Stack. Required only for Azure AD deployments.
    * **CloudAdminCredential**. The credential for the cloud administrator, necessary to access the privileged endpoint.
    * **AzCredential**. The credential for the Azure Stack service admin account. Use the same credentials that you used for deploying Azure Stack.

## Next steps

[Offer App Services as PaaS](azure-stack-app-service-overview.md)
