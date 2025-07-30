# 🛡️ SQL Server Express: Automated Backup Solution

This guide explains how to create an automated backup process for Microsoft SQL Server Express using:

1. A stored procedure (`sp_BackupDatabases`)
2. A batch file
3. Windows Task Scheduler

---

## 📌 Step 1: Create a Stored Procedure to Back Up Your Databases

### Connect to SQL Server:

1. Open **Microsoft SQL Server Management Studio (SSMS)**.
2. Connect to your instance using your **username and password**.  
   For example: `Lenovo\sqlexpress1029`.
3. Click on **New Query** in the toolbar.
4. Copy and paste the following code into the query window.


### 📄 Stored Procedure Code:

```sql
-- Copyright ? Microsoft Corporation.  All Rights Reserved. 
-- This code released under the terms of the 
-- Microsoft Public License (MS-PL, http://opensource.org/licenses/ms-pl.html.) 
USE [master]  
GO  
/****** Object:  StoredProcedure [dbo].[sp_BackupDatabases] ******/  
SET ANSI_NULLS ON  
GO  
SET QUOTED_IDENTIFIER ON  
GO  
-- =============================================  
-- Author: Microsoft  
-- Create date: 2010-02-06 
-- Description: Backup Databases for SQLExpress 
-- Parameter1: databaseName  
-- Parameter2: backupType F=full, D=differential, L=log 
-- Parameter3: backup file location 
-- ============================================= 
CREATE PROCEDURE [dbo].[sp_BackupDatabases]   
            @databaseName sysname = null, 
            @backupType CHAR(1), 
            @backupLocation nvarchar(200)  
AS  
       SET NOCOUNT ON;  
            DECLARE @DBs TABLE 
            ( 
                  ID int IDENTITY PRIMARY KEY, 
                  DBNAME nvarchar(500) 
            ) 
             -- Pick out only databases which are online in case ALL databases are chosen to be backed up 
             -- If specific database is chosen to be backed up only pick that out from @DBs 
            INSERT INTO @DBs (DBNAME) 
            SELECT Name FROM master.sys.databases 
            where state=0 
            AND name= ISNULL(@databaseName ,name)
            ORDER BY Name
            -- Filter out databases which do not need to backed up 
            IF @backupType='F' 
                  BEGIN 
                  DELETE @DBs where DBNAME IN ('tempdb','Northwind','pubs','AdventureWorks') 
                  END 
            ELSE IF @backupType='D' 
                  BEGIN 
                  DELETE @DBs where DBNAME IN ('tempdb','Northwind','pubs','master','AdventureWorks') 
                  END 
            ELSE IF @backupType='L' 
                  BEGIN 
                  DELETE @DBs where DBNAME IN ('tempdb','Northwind','pubs','master','AdventureWorks') 
                  END 
            ELSE 
                  BEGIN 
                  RETURN 
                  END 
            -- Declare variables 
            DECLARE @BackupName nvarchar(100) 
            DECLARE @BackupFile nvarchar(300) 
            DECLARE @DBNAME nvarchar(300) 
            DECLARE @sqlCommand NVARCHAR(1000)  
	        DECLARE @dateTime NVARCHAR(20) 
            DECLARE @Loop int                   
            -- Loop through the databases one by one 
            SELECT @Loop = min(ID) FROM @DBs 
      WHILE @Loop IS NOT NULL 
      BEGIN 
-- Database Names have to be in [dbname] format since some have - or _ in their name 
      SET @DBNAME = '['+(SELECT DBNAME FROM @DBs WHERE ID = @Loop)+']' 
-- Set the current date and time n yyyyhhmmss format 
      SET @dateTime = REPLACE(CONVERT(VARCHAR, GETDATE(),101),'/','') + '_' +  REPLACE(CONVERT(VARCHAR, GETDATE(),108),':','')   
-- Create backup filename in path\filename.extension format for full,diff and log backups 
      IF @backupType = 'F' 
            SET @BackupFile = @backupLocation+REPLACE(REPLACE(@DBNAME, '[',''),']','')+ '_FULL_'+ @dateTime+ '.BAK' 
      ELSE IF @backupType = 'D' 
            SET @BackupFile = @backupLocation+REPLACE(REPLACE(@DBNAME, '[',''),']','')+ '_DIFF_'+ @dateTime+ '.BAK' 
      ELSE IF @backupType = 'L' 
            SET @BackupFile = @backupLocation+REPLACE(REPLACE(@DBNAME, '[',''),']','')+ '_LOG_'+ @dateTime+ '.TRN' 
-- Provide the backup a name for storing in the media 
      IF @backupType = 'F' 
            SET @BackupName = REPLACE(REPLACE(@DBNAME,'[',''),']','') +' full backup for '+ @dateTime 
      IF @backupType = 'D' 
            SET @BackupName = REPLACE(REPLACE(@DBNAME,'[',''),']','') +' differential backup for '+ @dateTime 
      IF @backupType = 'L' 
            SET @BackupName = REPLACE(REPLACE(@DBNAME,'[',''),']','') +' log backup for '+ @dateTime 
-- Generate the dynamic SQL command to be executed 
       IF @backupType = 'F'  
                  BEGIN 
               SET @sqlCommand = 'BACKUP DATABASE ' +@DBNAME+  ' TO DISK = '''+@BackupFile+ ''' WITH INIT, NAME= ''' +@BackupName+''', NOSKIP, NOFORMAT' 
                  END 
       IF @backupType = 'D' 
                  BEGIN 
               SET @sqlCommand = 'BACKUP DATABASE ' +@DBNAME+  ' TO DISK = '''+@BackupFile+ ''' WITH DIFFERENTIAL, INIT, NAME= ''' +@BackupName+''', NOSKIP, NOFORMAT'         
                  END 
       IF @backupType = 'L'  
                  BEGIN 
               SET @sqlCommand = 'BACKUP LOG ' +@DBNAME+  ' TO DISK = '''+@BackupFile+ ''' WITH INIT, NAME= ''' +@BackupName+''', NOSKIP, NOFORMAT'         
                  END 
-- Execute the generated SQL command 
       EXEC(@sqlCommand) 
-- Goto the next database 
SELECT @Loop = min(ID) FROM @DBs where ID>@Loop 
END 
````

5. Execute the script.
   
### 📝 Customize Exclusions

To exclude a database from backup, add it to the exclusion list in this section:

```sql
IF @backupType='F' 
BEGIN 
    DELETE @DBs WHERE DBNAME IN ('tempdb','Northwind','pubs','AdventureWorks')
END
```

---

## 📌 Step 2: Create a Batch File

Create a batch file to execute the stored procedure using SQLCMD.

### 🪟 Instructions:

1. Open **Notepad** or any text editor.
2. Paste the following code:

```bat
sqlcmd -S .\SQLEXPRESS2019 -E -Q "EXEC sp_BackupDatabases @backupLocation='D:\SQLBackups\', @backupType='F'"
```

3. Save the file as:

```
sqlexpressfullbackup.bat
```

> ⚠️ Make sure to change:
>
> * `SQLEXPRESS2019` to match your SQL instance.
> * `D:\SQLBackups\` to your preferred backup folder.

> 🔐 If you're using **SQL Authentication**, protect this file — the password may be visible in plain text if included.

---

## 📌 Step 3: Schedule the Backup Using Task Scheduler

Use Windows Task Scheduler to automate the batch file daily.

### 🧭 Instructions:

1. Open **Start** → Type **Task Scheduler** and open it.
2. In the left panel, right-click **Task Scheduler Library** → select **Create Basic Task**.
3. Name it (e.g., `SQLBackup`) → click **Next**.
4. Choose **Daily** → click **Next**.
5. Set recurrence to **every 1 day** → click **Next**.
6. Select **Start a program** → click **Next**.
7. Click **Browse**, select `sqlexpressfullbackup.bat` → click **Open**.
8. Check: **"Open the Properties dialog for this task when I click Finish"** → click **Finish**.

### 🛠️ In Properties (Advanced Settings):

* Under **General**, select:

  * ✅ “Run whether user is logged on or not”
  * ✅ “Run with highest privileges”
* Under **Conditions**, uncheck:

  * 🔲 “Start the task only if the computer is on AC power” (optional for laptops)

---

## ✅ You're Done!

Your SQL Express databases will now back up automatically. Monitor the backup folder for `.BAK` or `.TRN` files to confirm success.

---
