# SQL Server Log Growth and Orphaned Instances

* [Overview](#overview)
* [1. Identifying the Root Cause](#1-identifying-the-root-cause)
* [2. Verifying the Active SQL Server Version](#2-verifying-the-active-sql-server-version)
* [3. Investigating Transaction Log Growth](#3-investigating-transaction-log-growth)
* [4. Fixing Log File Growth (Development Environment)](#4-fixing-log-file-growth-development-environment)

  * [Step 1 — Switch to SIMPLE Recovery](#step-1--switch-to-simple-recovery)
  * [Step 2 — Identify Log File Name](#step-2--identify-log-file-name)
  * [Step 3 — Shrink Log File (Emergency Use Only)](#step-3--shrink-log-file-emergency-use-only)
* [5. Detecting Multiple Installed Instances](#5-detecting-multiple-installed-instances)
* [6. Handling a Partially Removed SQL Server 2019 Instance](#6-handling-a-partially-removed-sql-server-2019-instance)

  * [Scenario A — Service Does NOT Exist](#scenario-a--service-does-not-exist)
  * [Scenario B — Service Still Exists](#scenario-b--service-still-exists)
* [7. Preventive Monitoring & Best Practices](#7-preventive-monitoring--best-practices)

  * [Choose the Correct Recovery Model](#1️⃣-choose-the-correct-recovery-model)
  * [Always Configure Log Backups (FULL Recovery)](#2️⃣-always-configure-log-backups-full-recovery)
  * [Set Sensible Autogrowth Settings](#3️⃣-set-sensible-autogrowth-settings)
  * [Monitor Disk Space](#4️⃣-monitor-disk-space)
  * [Avoid Installing SQL Server on C: (Production)](#5️⃣-avoid-installing-sql-server-on-c-production)
  * [Review Installed Instances Periodically](#6️⃣-review-installed-instances-periodically)
* [8. Final Results](#8-final-results)
* [Key Lessons](#key-lessons)





## Overview

A critical disk space issue was detected on the system drive (C:), where only ~2GB remained free out of 246GB.

Initial assumptions included:

* Docker images
* SDK installations
* Temporary files

However, deeper investigation revealed the real causes:

1. Uncontrolled transaction log growth in a development database
2. Multiple installed SQL Server instances
3. A partially removed (orphaned) SQL Server 2019 instance leaving large data files behind

This document explains the full investigation and resolution process.



## 1. Identifying the Root Cause

Using disk analysis tools such as:

* TreeSize
* WinDirStat

I identified the following directory consuming over 55GB:

```
C:\Program Files\Microsoft SQL Server
```

Two instances were installed:

```
MSSQL15.MSSQLSERVER  (~29GB)
MSSQL16.MSSQLSERVER  (~23GB)
```

* MSSQL16 → Active (SQL Server 2022)
* MSSQL15 → Older installation (SQL Server 2019)



## 2. Verifying the Active SQL Server Version

To confirm the running engine version:

```sql
SELECT @@VERSION;
```

Result:

```
SQL Server 2022 Developer Edition
```

This confirmed that the active instance was:

```
MSSQL16.MSSQLSERVER
```

Which corresponds to Microsoft SQL Server 2022.



## 3. Investigating Transaction Log Growth

I inspected the database recovery model:

```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'DbName';
```

Result:

```
DbName    FULL
```

The database was configured with **FULL recovery model**.

---

### Why This Caused Disk Growth

In FULL recovery mode:

* Transaction log files (.ldf) continue growing
* Log space is not reusable until a log backup occurs
* No log backup jobs were configured
* Result: Log file grew indefinitely

To check log space usage:

```sql
DBCC SQLPERF(LOGSPACE);
```

To inspect database files:

```sql
USE DbName;
GO

SELECT name, type_desc, size
FROM sys.database_files;
```

The log file had grown to several gigabytes and was nearly fully utilized.



## 4. Fixing Log File Growth (Development Environment)

Since this was a **local development database**, point-in-time recovery was not required.

> ⚠️ Important: In production environments, switching to SIMPLE recovery may not be acceptable if point-in-time restore is required.

---

### Step 1 — Switch to SIMPLE Recovery

```sql
ALTER DATABASE DbName SET RECOVERY SIMPLE;
GO
```

This enables automatic log truncation.

---

### Step 2 — Identify Log File Name

```sql
SELECT name 
FROM sys.database_files 
WHERE type_desc = 'LOG';
```

Example:

```
DbName_log
```

---

### Step 3 — Shrink Log File (Emergency Use Only)

```sql
DBCC SHRINKFILE (DbName_log, 512);
GO
```

This reduces the log file to approximately 512MB.

> ⚠️ Shrinking database files should only be used as a corrective action after abnormal growth — not as routine maintenance.



## 5. Detecting Multiple Installed Instances

To verify installed SQL Server instances, I inspected Windows Services:

1. Open **Run** → type `services.msc`
2. Locate services starting with:
```
SQL Server (...)
```

I identified the following entries:

```
SQL Server (MSSQLSERVER)
SQL Server (MSSQL15.MSSQLSERVER)
SQL Server (MSSQL16.MSSQLSERVER)
```

Findings:

* SQL Server 2022 was running
* SQL Server 2019 was stopped or partially removed

Multiple side-by-side installations of Microsoft SQL Server can silently consume large disk space if not reviewed periodically.



## 6. Handling a Partially Removed SQL Server 2019 Instance

The following directory was missing:

```
C:\Program Files\Microsoft SQL Server\150\Setup Bootstrap\SQLServer2019
```

This indicated a partial uninstall.

---

### Scenario A — Service Does NOT Exist

If:

```
SQL Server (MSSQL15.MSSQLSERVER)
```

is NOT listed in `services.msc`, then:

* The engine is unregistered
* Only data files remain on disk

Safe to delete:

```
C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER
```

Then restart the machine.

Recovered space: ~25–30GB.

---

### Scenario B — Service Still Exists

Run Command Prompt as Administrator:

```
sc delete MSSQL$MSSQLSERVER
```

(Adjust service name if necessary.)

Then manually remove the directory.


## 7. Preventive Monitoring & Best Practices

To avoid similar incidents:

### 1️⃣ Choose the Correct Recovery Model

* Development → SIMPLE
* Production → FULL (with scheduled log backups)

### 2️⃣ Always Configure Log Backups (FULL Recovery)

Schedule regular log backups via SQL Agent jobs.


**Using SQL Server Management Studio (SSMS) GUI:**

1. Open **SQL Server Agent → Jobs**
2. Right-click → **New Job…**
3. Add a **Step** with type **Transact-SQL script (T-SQL)**
4. Paste the `BACKUP LOG` command
   ```sql
   -- Perform a transaction log backup
   BACKUP LOG DbName
   TO DISK = 'D:\SQLBackups\DbName_log.trn'
   WITH INIT;
   ```
5. Schedule the job to run at regular intervals (e.g., every 15–30 minutes)



### 3️⃣ Set Sensible Autogrowth Settings

* Avoid unlimited autogrowth
* Use fixed growth sizes instead of percentage growth

💡 **How to apply this:**


**Using T-SQL:**

```sql
ALTER DATABASE DbName
MODIFY FILE (
    NAME = DbName_log,
    FILEGROWTH = 512MB,  -- fixed growth size
    MAXSIZE = 10GB       -- prevent unlimited growth
);
```


**Using SSMS GUI:**

1. Right-click the database → **Properties** → **Files**
2. Select the log file and click **Autogrowth…**
3. Choose **Growth in MB** (e.g., 512MB)
4. Set a **Maximum File Size** (e.g., 10GB)
5. Click **OK**



### 4️⃣ Monitor Disk Space

* Enable disk alerts
* Periodically review large directories
* Use tools like TreeSize proactively

### 5️⃣ Avoid Installing SQL Server on C: (Production)

In production environments:

* Install engine binaries separately
* Store data/log files on dedicated volumes

### 6️⃣ Review Installed Instances Periodically

Remove unused or legacy SQL Server instances to prevent orphaned data consumption.


## 8. Final Results

| Action                    | Space Freed  |
| ------------------------- | ------------ |
| Shrinking Log File        | ~5–6GB       |
| Removing MSSQL15 Instance | ~25–30GB     |
| **Total Freed**           | **~30–35GB** |

This resolved:

* Docker startup failures
* Windows performance degradation
* Disk full warnings
* Build instability



## Key Lessons

1. FULL recovery without log backups = guaranteed disk growth
2. SIMPLE recovery is appropriate for local development
3. Never store backups inside:

```
C:\Program Files\Microsoft SQL Server\...\DATA
```

4. Always verify active SQL instance before deleting directories
5. Disk analysis tools are essential in storage troubleshooting

