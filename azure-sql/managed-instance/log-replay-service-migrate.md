---
title: Migrate databases to SQL Managed Instance using Log Replay Service
description: Learn how to migrate databases from SQL Server to SQL Managed Instance by using Log Replay Service (LRS). 
services: sql-database
ms.service: sql-managed-instance
ms.subservice: migration
ms.custom: devx-track-azurecli, devx-track-azurepowershell
ms.topic: how-to
author: danimir
ms.author: danil
ms.reviewer: mathoma
ms.date: 07/15/2022
---

# Migrate databases from SQL Server to SQL Managed Instance by using Log Replay Service (Preview)
[!INCLUDE[appliesto-sqlmi](../includes/appliesto-sqlmi.md)]

This article explains how to manually configure database migration from SQL Server 2008-2019 to Azure SQL Managed Instance by using Log Replay Service (LRS), currently in public preview. LRS is a free of charge cloud service enabled for Azure SQL Managed Instance based on SQL Server log-shipping technology.

[Azure Database Migration Service](/azure/dms/tutorial-sql-server-to-managed-instance) and LRS use the same underlying migration technology and APIs. LRS further enables complex custom migrations and hybrid architectures between on-premises SQL Server and SQL Managed Instance.

## When to use Log Replay Service

When you can't use Azure Database Migration Service for migration, you can use LRS directly with PowerShell, Azure CLI cmdlets, or APIs to manually build and orchestrate database migrations to SQL Managed Instance. 

Consider using LRS in the following cases:
- You need more control for your database migration project.
- There's little tolerance for downtime during migration cutover.
- The Database Migration Service executable file can't be installed to your environment.
- The Database Migration Service executable file doesn't have file access to your database backups.
- No access to the host OS is available, or there are no administrator privileges.
- You can't open network ports from your environment to Azure.
- Network throttling, or proxy blocking issues exist in your environment.
- Backups are stored directly to Azure Blob Storage through the `TO URL` option.
- You need to use differential backups.

> [!NOTE]
> - We recommend automating the migration of databases from SQL Server to SQL Managed Instance by using Database Migration Service. Consider using LRS to orchestrate migrations when Database Migration Service doesn't fully support your scenarios.
> - LRS is the only method to restore differential backups on managed instance. It isn't possible to manually restore differential backups on managed instance, nor to manually set the `NORECOVERY` mode using T-SQL.
>

## How it works

Building a custom solution to migrate databases to the cloud with LRS requires several orchestration steps, as shown in the diagram and a table later in this section.

Migration consists of making database backups on SQL Server with `CHECKSUM` enabled, and copying backup files to Azure Blob Storage. Full, log, and differential backups are supported. LRS cloud service is used to restore backup files from Azure Blob Storage to SQL Managed Instance. Blob Storage serves as an intermediary storage between SQL Server and SQL Managed Instance.

LRS monitors Blob Storage for any new differential or log backups added after the full backup has been restored. LRS then automatically restores these new files. You can use the service to monitor the progress of backup files being restored to SQL Managed Instance, and stop the process if necessary.

LRS doesn't require a specific naming convention for backup files. It scans all files placed on Azure Blob Storage and constructs the backup chain from reading the file headers only. Databases are in a **restoring** state during the migration process. Databases are restored in [NORECOVERY](/sql/t-sql/statements/restore-statements-transact-sql#comparison-of-recovery-and-norecovery) mode, so they can't be used for read or write workloads until the migration process completes.

If you're migrating several databases, you need to:

- Place backup files for each database in a separate folder on Azure Blob Storage in a flat-file structure. For example, use separate database folders: `bolbcontainer/database1/files`, `blobcontainer/database2/files`, etc.
- Don't use nested folders inside database folders as this structure isn't supported. For example, don't use subfolders: `blobcontainer/database1/subfolder/files`.
- Start LRS separately for each database.
- Specify different URI paths to separate database folders on Azure Blob Storage. 

### Autocomplete versus Continuous mode migration

You can start LRS in either **autocomplete** or **continuous** mode.

Use autocomplete mode in cases when you have the entire backup chain generated in advance, and when you don't plan to add any more files once the migration has been started. This migration mode is recommended for passive workloads that don't require data catch-up. Upload all backup files to the Azure Blob Storage, and start the autocomplete mode migration. The migration will complete automatically when the last specified backup file has been restored. Migrated database will become available for read and write access on SQL Managed Instance.

In case that you plan to keep adding new backup files while migration is in progress, use continuous mode. This mode is recommended for active workloads requiring data catch-up. Upload the currently available backup chain to Azure Blob Storage, start the migration in continuous mode, and keep adding new backup files from your workload as needed. The system will periodically scan Azure Blob Storage folder and restore any new log or differential backup files found. When you're ready to cutover, stop the workload on your SQL Server, generate and upload the last backup file. Ensure that the last backup file has restored by watching that the final log-tail backup is shown as restored on SQL Managed Instance. Then, initiate manual cutover. The final cutover step makes the database come online and available for read and write access on SQL Managed Instance.

After LRS is stopped, either automatically through autocomplete, or manually through cutover, you can't resume the restore process for a database that was brought online on SQL Managed Instance. For example, once migration completes, you're no longer able to restore more differential backups for an online database. To restore more backup files after migration completes, you need to delete the database from the managed instance and restart the migration from the beginning. 

### Migration workflow

Typical migration workflow is shown in the image below, and steps outlined in the table.

Autocomplete mode needs to be used only when all backup chain files are available in advance. This mode is recommended for passive workloads for which no data catch-up is required.

Continuous mode migration needs to be used when you don't have the entire backup chain in advance, and when you plan to add new backup files once the migration is in progress. This mode is recommended for active workloads for which data catch-up is required.

:::image type="content" source="./media/log-replay-service-migrate/log-replay-service-conceptual.png" alt-text="Diagram that explains the Log Replay Service orchestration steps for SQL Managed Instance." border="false":::
	
| Operation | Details |
| :----------------------------- | :------------------------- |
| **1. Copy database backups from SQL Server to Blob Storage**. | Copy full, differential, and log backups from SQL Server to a Blob Storage container by using [AzCopy](/azure/storage/common/storage-use-azcopy-v10) or [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/). <br /><br />Use any file names. LRS doesn't require a specific file-naming convention.<br /><br />Use a separate folder for each database when migrating several databases. |
| **2. Start LRS in the cloud**. | You can start the service with PowerShell ([start-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/start-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_start cmdlets](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-start)). Choose between **autocomplete** or **continuous** migration modes. <br /><br /> Start LRS separately for each database that points to a backup folder on Blob Storage. <br /><br /> After the service starts, it will take backups from the Blob Storage container and start restoring them to SQL Managed Instance.<br /><br /> When started in autocomplete mode, LRS restores all backups until the specified last backup file. All backup files must be uploaded in advance, and it isn't possible to add any new backup files while migration is in progress. This mode is recommended for passive workloads for which no data catch-up is required. <br /><br /> When started in continuous mode, LRS restores all the  backups initially uploaded and then watches for any new files uploaded to the folder. The service will continuously apply logs based on the log sequence number (LSN) chain until it's stopped manually. This mode is recommended for active workloads for which data catch-up is required. |
| **2.1. Monitor the operation's progress**. | You can monitor progress of the restore operation with PowerShell ([get-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/get-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_show cmdlets](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-show)). |
| **2.2. Stop the operation if required (optional)**. | If you need to stop the migration process, use PowerShell ([stop-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/stop-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_stop](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-stop)). <br /><br /> Stopping the operation deletes the database that you're restoring to SQL Managed Instance. After you stop an operation, you can't resume LRS for a database. You need to restart the migration process from the beginning. |
| **3. Cut over to the cloud when you're ready**. | If LRS was started in autocomplete mode, the migration will automatically complete once the specified last backup file has been restored. <br /><br />  If LRS was started in continuous mode, stop the application and workload. Take the last log-tail backup and upload it to Azure Blob Storage. Ensure that the last log-tail backup has been restored on managed instance. Complete the cutover by initiating an LRS `complete` operation with PowerShell ([complete-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/complete-azsqlinstancedatabaselogreplay)) or the Azure CLI [az_sql_midb_log_replay_complete](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-complete). This operation stops LRS and brings the database online for read and write workloads on SQL Managed Instance. <br /><br /> Repoint the application connection string from SQL Server to SQL Managed Instance. You'll need to orchestrate this step yourself, either through a manual connection string change in your application, or automatically (for example, if your application can read the connection string from a property, or a database). |

### Migrating large databases

If migrating large databases of several terrabytes in size, consider the following:
- Single LRS job can run for a maximum of 30 days. On expiry of this timeframe, the job will be automatically canceled.
- In the case of long-running jobs, system updates will interrupt and prolong migration jobs. It's highly recommended to use [maintenance window]( ../database/maintenance-window.md) to schedule planned system updates. Plan your migration around the scheduled maintenance window.
- Migration jobs interrupted by system updates will be automatically suspended and resumed for General Purpose managed instances, and restarted for Business Critical managed instances. These updates will affect the timeframe of your migration.
- To increase the upload speed of your SQL Server backup files to Azure Blob Storage, provided there's a sufficient network bandwidth from your infrastructure, consider using parallelization with multiple threads.

## Getting started

Consider the requirements in this section to get started with using LRS to migrate. 

### SQL Server 

Make sure you have the following requirements for SQL Server: 

- SQL Server versions from 2008 to 2022
- Full backup of databases (one or multiple files)
- Differential backup (one or multiple files)
- Log backup (not split for a transaction log file)
- `CHECKSUM` enabled for backups (mandatory)

### Azure 

Make sure you have the following requirements for Azure: 

- PowerShell Az.SQL module version 2.16.0 or later ([installed](https://www.powershellgallery.com/packages/Az.Sql/) or accessed through [Azure Cloud Shell](/azure/cloud-shell/))
- Azure CLI version 2.19.0 or later ([installed](/cli/azure/install-azure-cli))
- Azure Blob Storage container provisioned
- Shared access signature (SAS) security token with read and list permissions generated for the Blob Storage container

### Azure RBAC permissions

Running LRS through the provided clients requires one of the following Azure roles:
- Subscription Owner role
- [SQL Managed Instance Contributor](/azure/role-based-access-control/built-in-roles#sql-managed-instance-contributor) role
- Custom role with the following permission: `Microsoft.Sql/managedInstances/databases/*`

## Requirements

Ensure the following requirements are met:
- Use the full recovery model on SQL Server (mandatory).
- Use `CHECKSUM` for backups on SQL Server (mandatory).
- Place backup files for an individual database inside a separate folder in a flat-file structure (mandatory). Nested folders inside database folders aren't supported.

## Best practices

We recommend the following best practices:
- Run [Data Migration Assistant](/sql/dma/dma-overview) to validate that your databases are ready to be migrated to SQL Managed Instance. 
- Split full and differential backups into multiple files, instead of using a single file.
- Enable backup compression to help the network transfer speeds.
- Use Cloud Shell to run PowerShell or CLI scripts, because it will always be updated to the latest cmdlets released.
- Configure [maintenance window](../database/maintenance-window.md) to allow scheduling of system updates at a specific day/time. This configuration will help achieve a more predictable time of database migrations, as impactful system upgrades interrupt migration in progress.
- Plan to complete a single LRS migration job within a maximum of 30 days. On expiry of this timeframe, the LRS job will be automatically canceled.

> [!IMPORTANT]
> - You can't use databases being restored through LRS until the migration process completes. 
> - LRS doesn't support read-only access to databases during the migration.
> - After the migration completes, the migration process is finalized and can't be resumed with additional differential backups.

> [!TIP]
> System updates on managed instance will take precedence over database migrations in progress. All pending LRS migrations in case of a system update on Managed Instance will be suspended and resumed once the update has been applied. This system behavior might prolong migration time, especially in cases of large databases. To achieve a predictable time of database migrations, consider configuring [maintenance window](../database/maintenance-window.md) allowing scheduling of system updates at a specific day/time, and consider running and completing migration jobs outside of the scheduled maintenance window day/time.
> 

## Steps to migrate

To migrate using LRS, follow the steps in this section. 

### Make database backups on SQL Server

You can make database backups on SQL Server by using either of the following options:

- Back up to the local disk storage, and then upload files to Azure Blob Storage, if your environment restricts direct backups to Blob Storage.
- Back up directly to Blob Storage with the `TO URL` option in Transact-SQL (T-SQL), if your environment and security procedures allow it. 

Set databases that you want to migrate to the full recovery model to allow log backups.

```SQL
-- To permit log backups, before the full database backup, modify the database to use the full recovery
USE master
ALTER DATABASE SampleDB
SET RECOVERY FULL
GO
```

To manually make full, differential, and log backups of your database to local storage, use the following sample T-SQL scripts. Ensure the `CHECKSUM` option is enabled, as it's mandatory for LRS.


The following example takes a full database backup to the local disk: 

```SQL
-- Take full database backup to local disk
BACKUP DATABASE [SampleDB]
TO DISK='C:\BACKUP\SampleDB_full.bak'
WITH INIT, COMPRESSION, CHECKSUM
GO
```

The following example takes a differential backup to the local disk: 

```sql
-- Take differential database backup to local disk
BACKUP DATABASE [SampleDB]
TO DISK='C:\BACKUP\SampleDB_diff.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM
GO
```

The following example takes a transaction log backup to the local disk: 

```sql
-- Take transactional log backup to local disk
BACKUP LOG [SampleDB]
TO DISK='C:\BACKUP\SampleDB_log.trn'
WITH COMPRESSION, CHECKSUM
GO
```

### Create a storage account

Azure Blob Storage is used as intermediary storage for backup files between SQL Server and SQL Managed Instance. To create a new storage account and a blob container inside the storage account, follow these steps:

1. [Create a storage account](/azure/storage/common/storage-account-create?tabs=azure-portal).
2. [Crete a blob container](/azure/storage/blobs/storage-quickstart-blobs-portal) inside the storage account.

### Copy backups from SQL Server to Blob Storage

When migrating databases to a managed instance by using LRS, you can use the following approaches to upload backups to Blob Storage:

- SQL Server native [BACKUP TO URL](/sql/relational-databases/backup-restore/sql-server-backup-to-url) 
- [AzCopy](/azure/storage/common/storage-use-azcopy-v10) or [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer) to upload backups to a blob container
- Storage Explorer in the Azure portal

> [!NOTE]
> To migrate multiple databases using the same Azure Blob Storage container, place all backup files of an individual database into a separate folder inside the container. Use flat-file structure for each database folder, as nested folders aren't supported.
> 

### Make backups from SQL Server directly to Blob Storage

If your corporate and network policies allow it, take backups from SQL Server directly to Blob Storage by using the SQL Server native [BACKUP TO URL](/sql/relational-databases/backup-restore/sql-server-backup-to-url) option. If you can use this option, you don't need to take backups to local storage and upload them to Blob Storage.

As the first step, this operation requires you to generate an SAS authentication token for Blob Storage, and then import the token to SQL Server. The second step is to make backups with the `TO URL` option in T-SQL. Ensure that all backups are made with the `CHEKSUM` option enabled.

For reference, the following sample code makes backups to Blob Storage. This example doesn't include instructions on how to import the SAS token. You can find detailed instructions, including how to generate and import the SAS token to SQL Server, in the tutorial [Use Azure Blob Storage with SQL Server](/sql/relational-databases/tutorial-use-azure-blob-storage-service-with-sql-server-2016#1---create-stored-access-policy-and-shared-access-storage). 

The following example takes a full database backup to a URL: 

```SQL
-- Take a full database backup to a URL
BACKUP DATABASE [SampleDB]
TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/SampleDB_full.bak'
WITH INIT, COMPRESSION, CHECKSUM
GO
```

The following example takes a differential database backup to a URL: 

```sql
-- Take a differential database backup to a URL
BACKUP DATABASE [SampleDB]
TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/SampleDB_diff.bak'  
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM
GO
```

The following example takes a transaction log backup to a URL: 

```sql
-- Take a transactional log backup to a URL
BACKUP LOG [SampleDB]
TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/SampleDB_log.trn'  
WITH COMPRESSION, CHECKSUM
```

### Migration of multiple databases

If migrating multiple databases using the same Azure Blob Storage container, you must place backup files for different databases in separate folders inside the container. All backup files for a single database must be placed in a flat-file structure inside a database folder, and the folders can't be nested, as it's not supported. 

Below is an example of folder structure inside Azure Blob Storage container required to migrate multiple databases using LRS.

```URI
-- Place all backup files for database 1 in a separate "database1" folder in a flat-file structure.
-- Don't use nested folders inside database1 folder.
https://<mystorageaccountname>.blob.core.windows.net/<containername>/<database1>/<all-database1-backup-files>

-- Place all backup files for database 2 in a separate "database2" folder in a flat-file structure.
-- Don't use nested folders inside database2 folder.
https://<mystorageaccountname>.blob.core.windows.net/<containername>/<database2>/<all-database2-backup-files>

-- Place all backup files for database 3 in a separate "database3" folder in a flat-file structure. 
-- Don't use nested folders inside database3 folder.
https://<mystorageaccountname>.blob.core.windows.net/<containername>/<database3>/<all-database3-backup-files>
```

### Generate a Blob Storage SAS authentication token for LRS

Azure Blob Storage is used as intermediary storage for backup files between SQL Server and SQL Managed Instance. Generate an SAS authentication token for LRS  with only list and read permissions. The token enables LRS to access Blob Storage and uses the backup files to restore them to SQL Managed Instance. 

Follow these steps to generate the token:

1. Open **Storage Explorer** from the Azure portal.
2. Expand **Blob Containers**.
3. Right-click the blob container and select **Get Shared Access Signature**.

   :::image type="content" source="./media/log-replay-service-migrate/lrs-sas-token-01.png" alt-text="Screenshot that shows selections for generating an S A S authentication token.":::

4. Select the time frame for token expiration. Ensure the token is valid during your migration.
5. Select the time zone for the token: UTC or your local time.
	
   > [!IMPORTANT]
   > The time zone of the token and your managed instance might mismatch. Ensure that the SAS token has the appropriate time validity, taking time zones into consideration. To account for time zone differences, set the validity time frame **FROM** well before your migration window starts, and the **TO** time frame well after you expect your migration to complete.

6. Select **Read** and **List** permissions only.

   > [!IMPORTANT]
   > Don't select any other permissions. If you do, LRS won't start. This security requirement is by-design.

7. Select **Create**.

   :::image type="content" source="./media/log-replay-service-migrate/lrs-sas-token-02.png" alt-text="Screenshot that shows selections for S A S token expiration, time zone, and permissions, along with the Create button.":::

The SAS authentication is generated with the time validity that you specified. You need the URI version of the token, as shown in the following screenshot.

:::image type="content" source="./media/log-replay-service-migrate/lrs-generated-uri-token.png" alt-text="Screenshot that shows an example of the U R I version of an S A S token.":::

   > [!NOTE]
   > Using SAS tokens created with permissions set through defining a [stored access policy](/rest/api/storageservices/define-stored-access-policy) isn't supported at this time. Follow the instructions in this article to manually specify **Read** and **List** permissions for the SAS token.

### Copy parameters from the SAS token

Before you use the SAS token to start LRS, you need to understand its structure. The URI of the generated SAS token consists of two parts separated with a question mark (`?`), as shown in this example:

:::image type="content" source="./media/log-replay-service-migrate/lrs-token-structure.png" alt-text="Example U R I for a generated S A S token for Log Replay Service." border="false":::

The first part, starting with `https://` until the question mark (`?`), is used for the `StorageContainerURI` parameter that's fed as the input to LRS. It gives LRS information about the folder where the database backup files are stored.

The second part, starting after the question mark (`?`) and going all the way until the end of the string, is the `StorageContainerSasToken` parameter. This part is the actual signed authentication token, which is valid during the specified time. This part doesn't necessarily need to start with `sp=` as shown in the example. Your case may differ.

Copy the parameters as follows:

1. Copy the first part of the token, starting from `https://` all the way until the question mark (`?`). Use it as the `StorageContainerUri` parameter in PowerShell or the Azure CLI when starting LRS.

   :::image type="content" source="./media/log-replay-service-migrate/lrs-token-uri-copy-part-01.png" alt-text="Screenshot that shows copying the first part of the token.":::

2. Copy the second part of the token, starting after the question mark (`?`) all the way until the end of the string. Use it as the `StorageContainerSasToken` parameter in PowerShell or the Azure CLI when starting LRS.

   :::image type="content" source="./media/log-replay-service-migrate/lrs-token-uri-copy-part-02.png" alt-text="Screenshot that shows copying the second part of the token.":::
   
> [!NOTE]
> Don't include the question mark (`?`) when you copy either part of the token.
> 

### Log in to Azure and select a subscription

Use the following PowerShell cmdlet to log in to Azure:

```powershell
Login-AzAccount
```

Select the appropriate subscription where your managed instance resides by using the following PowerShell cmdlet:

```powershell
Select-AzSubscription -SubscriptionId <subscription ID>
```

## Start the migration

You start the migration by starting LRS. You can start the service in either autocomplete or continuous mode. 

When you use autocomplete mode, the migration completes automatically when the last of the specified backup files have been restored. This option requires the entire backup chain to be available in advance, and uploaded to Azure Blob Storage. It doesn't allow adding new backup files while migration is in progress. This option requires the start command to specify the filename of the last backup file. This mode is recommended for passive workloads for which data catch-up isn't required.

When you use continuous mode, the service continuously scans Azure Blob Storage folder and restores any new backup files that keep getting added while migration is in progress. The migration completes only after the manual cutover has been requested. Continuous mode migration needs to be used when you don't have the entire backup chain in advance, and when you plan to add new backup files once the migration is in progress. This mode is recommended for active workloads for which data catch-up is required.

Plan to complete a single LRS migration job within a maximum of 30 days. On expiry of this timeframe, the LRS job will be automatically canceled.

> [!NOTE]
> When migrating multiple databases, LRS must be started separately for each database pointing to the full URI path of Azure Blob storage container and the individual database folder.
> 

### Start LRS in autocomplete mode

Ensure that the entire backup chain has been uploaded to Azure Blob Storage. This option doesn't allow new backup files to be added once the migration is in progress.

To start LRS in autocomplete mode, use PowerShell or Azure CLI commands. Specify the last backup file name by using the `-LastBackupName` parameter. After restore of the last specified backup file has completed, the service automatically initiates a cutover.

The following PowerShell example starts LRS in autocomplete mode: 

```PowerShell
Start-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "ResourceGroup01" `
	-InstanceName "ManagedInstance01" `
	-Name "ManagedDatabaseName" `
	-Collation "SQL_Latin1_General_CP1_CI_AS" `
	-StorageContainerUri "https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>" `
	-StorageContainerSasToken "sv=2019-02-02&ss=b&srt=sco&sp=rl&se=2023-12-02T00:09:14Z&st=2019-11-25T16:09:14Z&spr=https&sig=92kAe4QYmXaht%2Fgjocqwerqwer41s%3D" `
	-AutoCompleteRestore `
	-LastBackupName "last_backup.bak"
```

The following Azure CLI example starts LRS in autocomplete mode: 

```CLI
az sql midb log-replay start -g mygroup --mi myinstance -n mymanageddb -a --last-bn "backup.bak"
	--storage-uri "https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>"
	--storage-sas "sv=2019-02-02&ss=b&srt=sco&sp=rl&se=2023-12-02T00:09:14Z&st=2019-11-25T16:09:14Z&spr=https&sig=92kAe4QYmXaht%2Fgjocqwerqwer41s%3D"
```

> [!IMPORTANT]
> - Ensure that the entire backup chain has been uploaded to Azure Blob Storage prior to starting the migration in autocomplete mode. This mode doesn't allow new backup files to be added once the migration is in progress.
> - Ensure that you have specified the last backup file correctly, and that you have not uploaded more files after it. If the system detects more backup files beyond the last specified backup file, the migration will fail.
>

### Start LRS in continuous mode

Ensure that you've uploaded your initial backup chain to Azure Blob Storage.

The following PowerShell example starts LRS in continuous mode:

```PowerShell
Start-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "ResourceGroup01" `
	-InstanceName "ManagedInstance01" `
	-Name "ManagedDatabaseName" `
	-Collation "SQL_Latin1_General_CP1_CI_AS" -StorageContainerUri "https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>" `
	-StorageContainerSasToken "sv=2019-02-02&ss=b&srt=sco&sp=rl&se=2023-12-02T00:09:14Z&st=2019-11-25T16:09:14Z&spr=https&sig=92kAe4QYmXaht%2Fgjocqwerqwer41s%3D"
```

The following Azure CLI example starts LRS in continuous mode:

```CLI
az sql midb log-replay start -g mygroup --mi myinstance -n mymanageddb
	--storage-uri "https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>"
	--storage-sas "sv=2019-02-02&ss=b&srt=sco&sp=rl&se=2023-12-02T00:09:14Z&st=2019-11-25T16:09:14Z&spr=https&sig=92kAe4QYmXaht%2Fgjocqwerqwer41s%3D"
```

> [!IMPORTANT]
> Once LRS has been started in continuous mode, you'll be able to add new log and differential backups to Azure Blob Storage until the manual cutover. Once manual cutover has been initiated, no additional differential files can be added, nor restored.
> 

### Scripting the migration job

PowerShell and CLI clients that start LRS in continuous mode are synchronous. in this mode, PowerShell and CLI will wait for the API response to report on success or failure to start the job. 

During this wait, the command won't return control to the command prompt. If you're scripting the migration experience, and you need the LRS start command to give back control immediately to continue with rest of the script, you can run PowerShell as a background job with the `-AsJob` switch. For example:

```PowerShell
$lrsjob = Start-AzSqlInstanceDatabaseLogReplay <required parameters> -AsJob
```

When you start a background job, a job object returns immediately, even if the job takes an extended time to complete. You can continue to work in the session without interruption while the job runs. For details on running PowerShell as a background job, see the [PowerShell Start-Job](/powershell/module/microsoft.powershell.core/start-job#description) documentation.

Similarly, to start an Azure CLI command on Linux as a background process, use the ampersand (`&`) at the end of the LRS start command:

```CLI
az sql midb log-replay start <required parameters> &
```

## Monitor migration progress

To monitor migration progress through PowerShell, use the following command:

```PowerShell
Get-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "ResourceGroup01" `
	-InstanceName "ManagedInstance01" `
	-Name "ManagedDatabaseName"
```

To monitor migration progress through the Azure CLI, use the following command:

```CLI
az sql midb log-replay show -g mygroup --mi myinstance -n mymanageddb
```

## Stop the migration (optional)

If you need to stop the migration, use PowerShell or the Azure CLI. Stopping the migration deletes the restoring database on SQL Managed Instance, so resuming the migration won't be possible.

To stop the migration process through PowerShell, use the following command:

```PowerShell
Stop-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "ResourceGroup01" `
	-InstanceName "ManagedInstance01" `
	-Name "ManagedDatabaseName"
```

To stop the migration process through the Azure CLI, use the following command:

```CLI
az sql midb log-replay stop -g mygroup --mi myinstance -n mymanageddb
```

## Complete the migration (continuous mode)

If you started LRS in continuous mode, ensure that your application, and SQL Server workload have been stopped to prevent any new backup files from being generated. Ensure that the last backup from SQL Server has been uploaded to Azure Blob Storage. Monitor the restore progress on managed instance, ensuring that the last log-tail backup has been restored.

Once the last log-tail backup has been restored on managed instance, initiate the manual cutover to complete the migration. After the cutover has completed, the database will become available for read and write access on managed instance.

To complete the migration process in LRS continuous mode through PowerShell, use the following command:

```PowerShell
Complete-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "ResourceGroup01" `
-InstanceName "ManagedInstance01" `
-Name "ManagedDatabaseName" `
-LastBackupName "last_backup.bak"
```

To complete the migration process in LRS continuous mode through the Azure CLI, use the following command:

```CLI
az sql midb log-replay complete -g mygroup --mi myinstance -n mymanageddb --last-backup-name "backup.bak"
```

## Limitations

Consider the following limitations of LRS: 

- During the migration process, databases being migrated can't be used for read-only access on SQL Managed Instance.
- Configure [maintenance window](../database/maintenance-window.md) to allow scheduling of system updates at a specific day/time. Plan to run and complete migrations outside of the scheduled maintenance window.
- LRS requires databases on SQL Server to be backed up with the `CHECKSUM` option enabled.
- The SAS token that LRS uses must be generated for the entire Azure Blob Storage container, and it must have **Read** and **List** permissions only. For example, if you grant **Read**, **List** and **Write** permissions, LRS won't be able to start because of the extra **Write** permission.
- Using SAS tokens created with permissions set through defining a [stored access policy](/rest/api/storageservices/define-stored-access-policy) isn't supported. Follow the instructions in this article to manually specify **Read** and **List** permissions for the SAS token.
- Backup files containing % and $ characters in the file name can't be consumed by LRS. Consider renaming such file names.
- Backup files for different databases must be placed in separate folders on Blob Storage in a flat-file structure. Nested folders inside individual database folders aren't supported.
- If using autocomplete mode, the entire backup chain needs to be available in advance on Azure Blob Storage. It isn't possible to add new backup files in autocomplete mode. Use continuous mode if you need to add new backup files while migration is in progress.
- LRS must be started separately for each database pointing to the full URI path containing an individual database folder. 
- LRS can support up to 100 simultaneous restore processes per single managed instance.
- Single LRS job can run for the maximum of 30 days, after which it will be automatically canceled.

> [!TIP]
> If you require database to be R/O accessible during the migration, a much longer timeframe to perform the migration, and the best possible minimum downtime, consider the [link feature for Managed Instance](managed-instance-link-feature-overview.md) as a recommended migration solution in these cases.
>

## Troubleshooting

After you start LRS, use the monitoring cmdlet (PowerShell: `get-azsqlinstancedatabaselogreplay` or Azure CLI: `az_sql_midb_log_replay_show`) to see the status of the operation. If LRS fails to start after some time and you get an error, check for the most common issues:

- Does an existing database on SQL Managed Instance have the same name as the one you're trying to migrate from SQL Server? Resolve this conflict by renaming one of the databases.
- Was the database backup on SQL Server made via the `CHECKSUM` option?
- Are the permissions granted for the SAS **token Read** and **List** _only_?
- Did you copy the SAS token for LRS after the question mark (`?`), with content starting like this: `sv=2020-02-10...`? 
- Is the SAS token validity time applicable for the time window of starting and completing the migration? There might be mismatches due to the different time zones used for SQL Managed Instance and the SAS token. Try regenerating the SAS token and extending the token validity of the time window before and after the current date.
- Are the database name, resource group name, and managed instance name spelled correctly?
- If you started LRS in autocomplete mode, was a valid filename for the last backup file specified?

## Next steps

- Learn more about [migrating to SQL Managed Instance using the link feature](managed-instance-link-feature-overview.md).
- Learn more about [migrating from SQL Server to SQL Managed instance](../migration-guides/managed-instance/sql-server-to-managed-instance-guide.md).
- Learn more about [differences between SQL Server and SQL Managed Instance](transact-sql-tsql-differences-sql-server.md).
- Learn more about [best practices to cost and size workloads migrated to Azure](/azure/cloud-adoption-framework/migrate/azure-best-practices/migrate-best-practices-costs).
