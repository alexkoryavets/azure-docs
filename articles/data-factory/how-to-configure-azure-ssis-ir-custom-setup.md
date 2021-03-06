---
title: "Customize setup for the Azure-SSIS integration runtime | Microsoft Docs"
description: "This article describes how to use the custom setup interface for the Azure-SSIS integration runtime to install additional components or change settings"
services: data-factory
documentationcenter: ""
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: conceptual
ms.date: 10/31/2018
author: swinarko
ms.author: sawinark
ms.reviewer: douglasl
manager: craigg
---
# Customize setup for the Azure-SSIS integration runtime

The custom setup interface for the Azure-SSIS Integration Runtime provides an interface to add your own setup steps during the provisioning or reconfiguration of your Azure-SSIS IR. Custom setup lets you alter the default operating configuration or environment (for example, to start additional Windows services or persist access credentials for file shares) or install additional components (for example, assemblies, drivers, or extensions) on each node of your Azure-SSIS IR.

You configure your custom setup by preparing a script and its associated files, and uploading them into a blob container in your Azure Storage account. You provide a Shared Access Signature (SAS) Uniform Resource Identifier (URI) for your container when you provision or reconfigure your Azure-SSIS IR. Each node of your Azure-SSIS IR then downloads the script and its associated files from your container and runs your custom setup with elevated privileges. When custom setup is finished, each node uploads the standard output of execution and other logs into your container.

You can install both free or unlicensed components, and paid or licensed components. If you're an ISV, see [How to develop paid or licensed components for the Azure-SSIS IR](how-to-develop-azure-ssis-ir-licensed-components.md).


## Current limitations

-   If you want to use `gacutil.exe` to install assemblies in the Global Assembly Cache (GAC), you need to provide `gacutil.exe` as part of your custom setup, or use the copy provided in the Public Preview container.

-   If you want to reference a subfolder in your script, `msiexec.exe` does not support the `.\` notation to reference the root folder. Use a command like `msiexec /i "MySubfolder\MyInstallerx64.msi" ...` instead of `msiexec /i ".\MySubfolder\MyInstallerx64.msi" ...`.

-   If you need to join your Azure-SSIS IR with custom setup to a virtual network, only Azure Resource Manager virtual network is supported. Classic virtual network is not supported.

-   Administrative share is currently not supported on the Azure-SSIS IR.

## Prerequisites

To customize your Azure-SSIS IR, you need the following things:

-   [Azure subscription](https://azure.microsoft.com/)

-   [An Azure SQL Database or Managed Instance server](https://ms.portal.azure.com/#create/Microsoft.SQLServer)

-   [Provision your Azure-SSIS IR](https://docs.microsoft.com/azure/data-factory/tutorial-deploy-ssis-packages-azure)

-   [An Azure Storage account](https://azure.microsoft.com/services/storage/). For custom setup, you upload and store your custom setup script and its associated files in a blob container. The custom setup process also uploads its execution logs to the same blob container.

## Instructions

1.  Download and install [Azure PowerShell](https://github.com/Azure/azure-powershell/releases/tag/v5.5.0-March2018) (version 5.4 or later).

1.  Prepare your custom setup script and its associated files (for example, .bat, .cmd, .exe, .dll, .msi, or .ps1 files).

    1.  You must have a script file named `main.cmd`, which is the entry point of your custom setup.

    1.  If you want additional logs generated by other tools (for example, `msiexec.exe`) to be uploaded into your container, specify the predefined environment variable, `CUSTOM_SETUP_SCRIPT_LOG_DIR` as the log folder in your scripts (for example,  `msiexec /i xxx.msi /quiet /lv %CUSTOM_SETUP_SCRIPT_LOG_DIR%\install.log`).

1.  Download, install, and launch [Azure Storage Explorer](http://storageexplorer.com/).

    1.  Under **(Local and Attached)**, right-select **Storage Accounts** and select **Connect to Azure storage**.

       ![Connect to Azure storage](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image1.png)

    1.  Select **Use a storage account name and key** and select **Next**.

       ![Use a storage account name and key](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image2.png)

    1.  Enter your Azure Storage account name and key, select **Next**, and then select **Connect**.

       ![Provide store account name and key](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image3.png)

    1.  Under your connected Azure Storage account, right-click on **Blob Containers**, select **Create Blob Container**, and name the new container.

       ![Create a blob container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image4.png)

    1.  Select the new container and upload your custom setup script and its associated files. Make sure that you upload `main.cmd` at the top level of the container, not in any folder. 

       ![Upload files to the blob container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image5.png)

    1.  Right-click the container and select **Get Shared Access Signature**.

       ![Get the Shared Access Signature for the container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image6.png)

    1.  Create the SAS URI for your container with a sufficiently long expiry time and with read + write + list permissions. You need the SAS URI to download and run your custom setup script and its associated files whenever any node of your Azure-SSIS IR is reimaged/restarted. You need write permission to upload setup execution logs.

        > [!IMPORTANT]
        > Please ensure that the SAS URI does not expire and custom setup resources are always available during the whole lifecycle of your Azure-SSIS IR, from creation to deletion, especially if you regularly stop and start your Azure-SSIS IR  during this period.

       ![Generate the Shared Access Signature for the container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image7.png)

    1.  Copy and save the SAS URI of your container.

       ![Copy and save the Shared Access Signature](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image8.png)

    1.  When you provision or reconfigure your Azure-SSIS IR with Data Factory UI, before you start your Azure-SSIS IR, enter the SAS URI of your container in the appropriate field on **Advanced Settings** panel:

       ![Enter the Shared Access Signature](media/tutorial-create-azure-ssis-runtime-portal/advanced-settings.png)

       When you provision or reconfigure your Azure-SSIS IR with PowerShell, before you start your Azure-SSIS IR, run the `Set-AzureRmDataFactoryV2IntegrationRuntime` cmdlet with the SAS URI of your container as the value for new `SetupScriptContainerSasUri` parameter. For example:

       ```powershell
       Set-AzureRmDataFactoryV2IntegrationRuntime -DataFactoryName $MyDataFactoryName `
                                                  -Name $MyAzureSsisIrName `
                                                  -ResourceGroupName $MyResourceGroupName `
                                                  -SetupScriptContainerSasUri $MySetupScriptContainerSasUri

       Start-AzureRmDataFactoryV2IntegrationRuntime -DataFactoryName $MyDataFactoryName `
                                                    -Name $MyAzureSsisIrName `
                                                    -ResourceGroupName $MyResourceGroupName
       ```

    1.  After your custom setup finishes and your Azure-SSIS IR starts, you can find the standard output of `main.cmd` and other execution logs in the `main.cmd.log` folder of your storage container.

1.  To see other custom setup examples, connect to the Public Preview container with Azure Storage Explorer.

    a.  Under **(Local and Attached)**, right-click **Storage Accounts**, select **Connect to Azure storage**, select **Use a connection string or a shared access signature URI**, and then select **Next**.

       ![Connect to Azure storage with the Shared Access Signature](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image9.png)

    b.  Select **Use a SAS URI** and enter the following SAS URI for the Public Preview container. Select **Next**, and the select **Connect**.

       `https://ssisazurefileshare.blob.core.windows.net/publicpreview?sp=rl&st=2018-04-08T14%3A10%3A00Z&se=2020-04-10T14%3A10%3A00Z&sv=2017-04-17&sig=mFxBSnaYoIlMmWfxu9iMlgKIvydn85moOnOch6%2F%2BheE%3D&sr=c`

       ![Provide the Shared Access Signature for the container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image10.png)

    c. Select the connected Public Preview container and double-click the `CustomSetupScript` folder. In this folder are the following items:

       1. A `Sample` folder, which contains a custom setup to install a basic task on each node of your Azure-SSIS IR. The task does nothing but sleep for a few seconds. The folder also contains a `gacutil` folder, which contains `gacutil.exe`. Additionally, `main.cmd` contains comments to persist access credentials for file shares.

       1. A `UserScenarios` folder, which contains several custom setups for real user scenarios.

    ![Contents of the public preview container](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image11.png)

    d. Double-click the `UserScenarios` folder. In this folder are the following items:

       1. A `.NET FRAMEWORK 3.5` folder, which contains a custom setup to install an earlier version of the .NET Framework that might be required for custom components on each node of your Azure-SSIS IR.

       1. An `AAS` folder, which contains a custom setup to install client libraries on each node of your Azure-SSIS IR that enable your Analysis Services tasks to connect to Azure Analysis Services (AAS) instance using service principal authentication. First, download the latest **MSOLAP (amd64)** and **AMO** client libraries/Windows installers - for example, `x64_15.0.900.108_SQL_AS_OLEDB.msi` and `x64_15.0.900.108_SQL_AS_AMO.msi` - from [here](https://docs.microsoft.com/azure/analysis-services/analysis-services-data-providers), then upload them all together with `main.cmd` into your container.  

       1. A `BCP` folder, which contains a custom setup to install SQL Server command-line utilities (`MsSqlCmdLnUtils.msi`), including the bulk copy program (`bcp`), on each node of your Azure-SSIS IR.

       1. An `EXCEL` folder, which contains a custom setup to install open-source assemblies (`DocumentFormat.OpenXml.dll`, `ExcelDataReader.DataSet.dll`, and `ExcelDataReader.dll`) on each node of your Azure-SSIS IR.

       1. An `MSDTC` folder, which contains a custom setup to modify the network and security configurations for the Microsoft Distributed Transaction Coordinator (MSDTC) service on each node of your Azure-SSIS IR. To ensure that MSDTC is started, please add Execute Process Task at the beginning of control flow in your packages to execute the following command: `%SystemRoot%\system32\cmd.exe /c powershell -Command "Start-Service MSDTC"` 

       1. An `ORACLE ENTERPRISE` folder, which contains a custom setup script (`main.cmd`) and silent install config file (`client.rsp`) to install the Oracle connectors and OCI driver on each node of your Azure-SSIS IR Enterprise Edition. This setup lets you use the Oracle Connection Manager, Source, and Destination. First, download Microsoft Connectors v5.0 for Oracle (`AttunitySSISOraAdaptersSetup.msi` and `AttunitySSISOraAdaptersSetup64.msi`) from [Microsoft Download Center](https://www.microsoft.com/en-us/download/details.aspx?id=55179) and the latest Oracle client - for example, `winx64_12102_client.zip` - from [Oracle](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/database12c-win64-download-2297732.html), then upload them all together with `main.cmd` and `client.rsp` into your container. If you use TNS to connect to Oracle, you also need to download `tnsnames.ora`, edit it, and upload it into your container, so it can be copied into the Oracle installation folder during setup.

       1. An `ORACLE STANDARD` folder, which contains a custom setup script (`main.cmd`) to install the Oracle ODP.NET driver on each node of your Azure-SSIS IR. This setup lets you use the ADO.NET Connection Manager, Source, and Destination. First, download the latest Oracle ODP.NET driver - for example, `ODP.NET_Managed_ODAC122cR1.zip` - from [Oracle](http://www.oracle.com/technetwork/database/windows/downloads/index-090165.html), and then upload it together with `main.cmd` into your container.

       1. An `SAP BW` folder, which contains a custom setup script (`main.cmd`) to install the SAP .NET connector assembly (`librfc32.dll`) on each node of your Azure-SSIS IR Enterprise Edition. This setup lets you use the SAP BW Connection Manager, Source, and Destination. First, upload the 64-bit or the 32-bit version of `librfc32.dll` from the SAP installation folder into your container, together with `main.cmd`. The script then copies the SAP assembly into the `%windir%\SysWow64` or `%windir%\System32` folder during setup.

       1. A `STORAGE` folder, which contains a custom setup to install Azure PowerShell on each node of your Azure-SSIS IR. This setup lets you deploy and run SSIS packages that run [PowerShell scripts to manipulate your Azure Storage account](https://docs.microsoft.com/azure/storage/blobs/storage-how-to-use-blobs-powershell). Copy `main.cmd`, a sample `AzurePowerShell.msi` (or install the latest version), and `storage.ps1` to your container. Use PowerShell.dtsx as a template for your packages. The package template combines an [Azure Blob Download Task](https://docs.microsoft.com/sql/integration-services/control-flow/azure-blob-download-task), which downloads `storage.ps1` as a modifiable PowerShell script, and an [Execute Process Task](https://blogs.msdn.microsoft.com/ssis/2017/01/26/run-powershell-scripts-in-ssis/)  that executes the script on each node.

       1. A `TERADATA` folder, which contains a custom setup script (`main.cmd`), its associated file (`install.cmd`), and installer packages (`.msi`). These files install Teradata connectors, the TPT API, and the ODBC driver on each node of your Azure-SSIS IR Enterprise Edition. This setup lets you use the Teradata Connection Manager, Source, and Destination. First, download the Teradata Tools and Utilities (TTU) 15.x zip file (for example,  `TeradataToolsAndUtilitiesBase__windows_indep.15.10.22.00.zip`) from [Teradata](http://partnerintelligence.teradata.com), and then upload it together with the above `.cmd` and `.msi` files into your container.

    ![Folders in the user scenarios folder](media/how-to-configure-azure-ssis-ir-custom-setup/custom-setup-image12.png)

    e. To try these custom setup samples, copy and paste the content from the selected folder into your container. When you provision or reconfigure your Azure-SSIS IR with PowerShell, run the `Set-AzureRmDataFactoryV2IntegrationRuntime` cmdlet with the SAS URI of your container as the value for new `SetupScriptContainerSasUri` parameter.

## Next steps

-   [Enterprise Edition of the Azure-SSIS Integration Runtime](how-to-configure-azure-ssis-ir-enterprise-edition.md)

-   [How to develop paid or licensed custom components for the Azure-SSIS integration runtime](how-to-develop-azure-ssis-ir-licensed-components.md)
