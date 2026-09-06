# Azure SQL Multi-Terabyte PROD-to-UAT Migration Using Active Geo-Replication

> A practical Azure architecture case study for creating an independent UAT copy of a multi-terabyte Azure SQL Database across subscriptions using temporary Active Geo-Replication.

![Microsoft Azure](https://img.shields.io/badge/Microsoft-Azure-0078D4?logo=microsoftazure)
![Azure SQL](https://img.shields.io/badge/Azure-SQL%20Database-0078D4)
![Architecture](https://img.shields.io/badge/Cloud-Architecture-blue)
![Status](https://img.shields.io/badge/Project-Completed-success)

---

## 📌 Overview

This repository documents a real-world Azure SQL Database migration pattern used to create an independent **UAT copy of a multi-terabyte production database** across separate Azure subscriptions.

The challenge was to move a very large Azure SQL Database from a **PROD Landing Zone** into an isolated **UAT Subscription** without impacting the live production database.

Traditional export/import approaches were not practical for the database size and operational requirements.

The solution was to use a temporary production database clone together with **Azure SQL Active Geo-Replication**.

The overall migration workflow was:

**Clone → Replicate → Validate → Detach → Validate Recovery → Remove Temporary Resources**

> **Security Notice**
>
> All names, subscriptions, IP addresses, resource groups, tenant information, database names, server names, and organization-specific identifiers in this repository have been intentionally anonymized.
>
> The architecture uses generic names such as `PROD-SQL`, `PROD-DB`, `UAT-SQL`, and `UAT-DB`.

---

# 🎯 Migration Objective

The requirement was to create an independent UAT database containing a copy of production data while ensuring that:

- The live production database remained unaffected.
- PROD and UAT remained isolated in separate Azure subscriptions.
- The migration supported a multi-terabyte database.
- UAT became an independent read/write database.
- No permanent replication dependency remained between PROD and UAT.
- Point-in-Time Restore (PITR) was available in UAT.
- Temporary production resources were removed after successful validation.
- Unnecessary Azure cost was eliminated after migration.

---

# 🏗️ High-Level Architecture

## Initial State

```text
                  AZURE PLATFORM
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
 ┌──────────────────┐        ┌──────────────────┐
 │ PROD LANDING ZONE│        │ UAT LANDING ZONE│
 │                  │        │                  │
 │ PROD Subscription│        │ UAT Subscription │
 │                  │        │                  │
 │    PROD-SQL      │        │     UAT-SQL      │
 │       │          │        │                  │
 │       ▼          │        │                  │
 │    PROD-DB       │        │                  │
 │                  │        │                  │
 └──────────────────┘        └──────────────────┘
The objective was to create an independent database inside the UAT environment without directly modifying the production database.


🔐 Design Principle — Protect the Production Database

Instead of configuring the migration directly against the live production database, a temporary clone was created.
PROD-SQL
   │
   ├── PROD-DB
   │      │
   │      │ Create temporary copy
   │      ▼
   └── PROD-DB-CLONE

This provided an additional isolation layer between the migration activity and the live production workload.

The temporary clone became the source for geo-replication.

🔄 Migration Architecture
                PROD LANDING ZONE
       ┌─────────────────────────────┐
       │                             │
       │      PROD Subscription      │
       │                             │
       │          PROD-SQL           │
       │             │               │
       │       ┌─────┴─────┐         │
       │       │           │         │
       │       ▼           ▼         │
       │   PROD-DB    PROD-DB-CLONE  │
       │                   │         │
       └───────────────────┼─────────┘
                           │
                           │
                  Active Geo-Replication
                           │
                           ▼
       ┌─────────────────────────────┐
       │                             │
       │       UAT Subscription      │
       │                             │
       │          UAT-SQL            │
       │             │               │
       │             ▼               │
       │           UAT-DB            │
       │                             │
       └─────────────────────────────┘
                UAT LANDING ZONE

The production clone acted as the temporary primary replica.

The UAT database acted as the secondary replica during synchronization.

❓ Why Not Use Traditional Backup and Restore?

Azure SQL Database differs from SQL Server running on an Azure VM or on-premises infrastructure.

Traditional SQL Server commands such as:

BACKUP DATABASE [PROD-DB]
TO DISK = 'backup.bak';

are not the normal native backup workflow for Azure SQL Database.

Azure SQL Database provides platform-managed automated backups and Point-in-Time Restore.

For smaller databases, a BACPAC export/import workflow may also be considered.

However, logical export/import can become operationally challenging for multi-terabyte databases because of factors such as:

Database size
Export duration
Import duration
Storage requirements
Long-running operations
Schema/data processing overhead
Operational migration windows

For this scenario, temporary Active Geo-Replication provided a more suitable mechanism for transferring the database.

🚀 Migration Workflow

The complete workflow consisted of the following phases:

Create PROD Clone
       │
       ▼
Configure Geo-Replication
       │
       ▼
Synchronize Database
       │
       ▼
Validate Replication
       │
       ▼
Stop Replication
       │
       ▼
Validate UAT Read/Write
       │
       ▼
Validate Replication Removal
       │
       ▼
Wait for PITR Restore Point
       │
       ▼
Validate UAT Data/Application
       │
       ▼
Delete Temporary PROD Clone
       │
       ▼
Migration Complete

1️⃣ Create Temporary PROD Clone

A temporary copy of the production database was created:
PROD-DB
   │
   ▼
PROD-DB-CLONE

The original production database remained untouched by the subsequent replication workflow.

The clone was sized appropriately to support the database workload and replication process.

2️⃣ Configure Active Geo-Replication

Azure SQL Active Geo-Replication was configured between:

Source

PROD Subscription
      │
      └── PROD-SQL
             │
             └── PROD-DB-CLONE


Target

UAT Subscription
      │
      └── UAT-SQL
             │
             └── UAT-DB

The UAT database was initially created as a readable secondary.

3️⃣ Monitor Replication

Creating the replica was not considered sufficient evidence that the database was ready to detach.

Replication was validated from the SQL Database engine.

The following Dynamic Management View was used:

SELECT
    partner_server,
    partner_database,
    role_desc,
    replication_state_desc,
    replication_lag_sec,
    last_replication,
    last_commit
FROM sys.dm_geo_replication_link_status;

The important state on the primary was:

role_desc                PRIMARY
replication_state_desc   CATCH_UP
replication_lag_sec      0

CATCH_UP indicated that the secondary had reached a transactionally consistent state and continued synchronizing changes from the primary.

Replication lag was also reviewed before proceeding.

4️⃣ Validate the Secondary

The UAT database was independently checked before breaking the replication relationship.

During replication, its role was expected to be:

SECONDARY

and replication state:

CATCH_UP

This provided confirmation from both sides of the replication topology.

5️⃣ Stop Replication

Once synchronization had been validated, the replication relationship was intentionally stopped.

The objective was not a disaster recovery failover.

The requirement was:

PROD-DB-CLONE
       │
       │  STOP REPLICATION
       X
       │
       ▼
UAT-DB
Standalone Database

Therefore:

Stop Replication was selected instead of:

Planned Failover
Forced Failover
Standby conversion

Stopping replication converted the UAT secondary into an independent Azure SQL Database.

6️⃣ Validate UAT Database State

After stopping replication, the UAT database was checked using:
SELECT
    DB_NAME() AS DatabaseName,
    DATABASEPROPERTYEX(DB_NAME(), 'Status') AS DatabaseStatus,
    DATABASEPROPERTYEX(DB_NAME(), 'Updateability') AS Updateability,
    DATABASEPROPERTYEX(DB_NAME(), 'UserAccess') AS UserAccess;

The expected result was:
DatabaseStatus : ONLINE
Updateability  : READ_WRITE
UserAccess     : MULTI_USER

This confirmed that UAT was now an independent, writable database.

7️⃣ Verify Replication Was Removed

The geo-replication DMV was checked again:
SELECT *
FROM sys.dm_geo_replication_link_status;

The expected result after successful removal was:
0 rows

This confirmed that no active geo-replication relationship remained.

The architecture had now changed to:
PROD-DB-CLONE              UAT-DB

Standalone                 Standalone
PROD                       UAT
   │                         │
   └──── No Replication ─────┘

8️⃣ Validate Point-in-Time Restore

The production clone was not deleted immediately.

This was an important safety decision.

After becoming standalone, the UAT database initially did not yet show an available Point-in-Time Restore point.

The backup policy was therefore checked.

Example configuration:

PITR Retention:
7 Days

Differential Backup Frequency:
12 Hours

Azure SQL Database automatically manages its backup infrastructure.

The migration process waited until the UAT database displayed an:
Earliest PITR Restore Point

before removing the temporary production source.

🛡️ Recovery Validation

The UAT database was considered ready for final cutover only after the following conditions were satisfied:

Database ONLINE                ✔
READ_WRITE                     ✔
MULTI_USER                     ✔
Replication removed            ✔
PITR policy configured         ✔
Restore point available        ✔
Data validation completed      ✔
Application connectivity       ✔

This is an important migration principle:

A database being online does not automatically mean the migration is complete. Recovery capability must also be validated.

9️⃣ Application and Data Validation

Before deleting the temporary source database, application and database checks should be performed.

Examples include:

SELECT COUNT(*)
FROM ImportantTable;

Latest transaction validation:

SELECT MAX(TransactionDate)
FROM ImportantTransactionTable;

Additional checks may include:

Critical table row counts
Recent transaction timestamps
Stored procedures
Views
Database users
Application connectivity
Application smoke tests
Query performance
Required permissions
Database configuration

The exact validation should be customized for the application.

🔟 Delete Temporary PROD Clone

After confirming:

UAT Database Online            ✔
UAT Database Read/Write        ✔
Replication Removed            ✔
PITR Available                 ✔
Application Validation         ✔
Data Validation                ✔

the temporary production clone was removed.

PROD-DB-CLONE
      │
      ▼
   DELETED

This eliminated the ongoing compute and storage cost associated with the temporary database.

The original production database remained untouched.

🏁 Final Architecture
             FINAL ARCHITECTURE


      PROD LANDING ZONE
┌──────────────────────────┐
│                          │
│    PROD Subscription     │
│                          │
│        PROD-SQL          │
│           │              │
│           ▼              │
│        PROD-DB           │
│                          │
│   PROD-DB-CLONE          │
│       DELETED            │
│                          │
└──────────────────────────┘


       NO REPLICATION
             │
             X
             │


       UAT LANDING ZONE
┌──────────────────────────┐
│                          │
│     UAT Subscription     │
│                          │
│         UAT-SQL          │
│            │             │
│            ▼             │
│          UAT-DB          │
│                          │
│        ONLINE            │
│        READ_WRITE        │
│        MULTI_USER        │
│        PITR ENABLED      │
│        STANDALONE        │
│                          │
└──────────────────────────┘
🔐 Network and Security Considerations

For enterprise deployments, SQL connectivity should follow private-access principles wherever possible.

A typical architecture can include:

Application / Management VM
           │
           ▼
      Private Network
           │
           ▼
     Private Endpoint
           │
           ▼
        UAT-SQL
           │
           ▼
        UAT-DB

Recommended controls include:

Azure Private Endpoint
Private DNS
Network isolation
Microsoft Entra authentication where appropriate
Least-privilege RBAC
SQL auditing
Microsoft Defender for SQL
Azure Monitor
Log Analytics
Key Vault for secrets
Restricted public network access

No private IP addresses or internal DNS information are included in this repository.

🌐 Private Endpoint DNS Consideration

When Azure SQL Database is accessed through a Private Endpoint, applications should connect using the SQL server's DNS name rather than directly using the Private Endpoint IP address.

Conceptually:

UAT-SQL.database.windows.net
            │
            ▼
     Private DNS Resolution
            │
            ▼
      Private Endpoint
            │
            ▼
         UAT-SQL

Using the DNS hostname is important for TLS certificate validation.

For enterprise environments, DNS should be managed centrally rather than relying on workstation or server host-file entries.

💰 Cost Optimization

Temporary resources can become expensive when working with multi-terabyte databases and high-vCore configurations.

The architecture therefore treated the production clone as an ephemeral migration resource.

Create Clone
     │
     ▼
Replicate
     │
     ▼
Validate
     │
     ▼
Detach
     │
     ▼
Verify Recovery
     │
     ▼
Delete Clone

Once UAT was independently recoverable, the temporary production clone was deleted.

This avoided paying indefinitely for two large database copies in the production environment.

🔄 Rollback Strategy

Before replication is stopped, rollback is straightforward:

Issue detected
      │
      ▼
Do NOT stop replication
      │
      ▼
Investigate / Correct

After replication has been stopped but before the PROD clone is deleted:

PROD-DB-CLONE    UAT-DB
      │             │
      │             └── Independent UAT copy
      │
      └── Temporary source still available

This intermediate period provides an additional safety window.

The temporary source should only be deleted after UAT validation and recovery validation have succeeded.

⚠️ Important Operational Considerations

This architecture should not be applied blindly to every Azure SQL migration.

Before implementation, evaluate:

Database size
Service tier
vCore requirements
Subscription permissions
Network connectivity
Private Endpoint configuration
DNS architecture
Replication compatibility
Backup retention requirements
RPO/RTO requirements
Application downtime requirements
Data sensitivity
Cost
Organizational change-management procedures

For production environments, always test the procedure in a controlled environment before executing a critical migration.

📊 Migration Decision Flow
Need PROD data in UAT?
          │
          ▼
Is database relatively small?
      │          │
     YES         NO
      │          │
      ▼          ▼
Consider      Evaluate scalable
BACPAC /      migration methods
Copy          such as replication
                 │
                 ▼
        Protect live PROD DB
                 │
                 ▼
        Create temporary clone
                 │
                 ▼
        Configure replication
                 │
                 ▼
          Validate CATCH_UP
                 │
                 ▼
          Stop replication
                 │
                 ▼
        Validate READ_WRITE
                 │
                 ▼
          Validate PITR
                 │
                 ▼
          Validate application
                 │
                 ▼
        Remove temporary clone
💡 Key Lessons Learned
1. Database size changes the migration strategy

A method suitable for a small database may not be appropriate for a multi-terabyte workload.

Migration architecture should be selected according to:

Database size
Available migration window
Network architecture
Recovery requirements
Operational risk
2. Protect the live production database

Using a temporary production clone isolated the live production database from the replication workflow.

PROD-DB
   │
   └── Protected

PROD-DB-CLONE
   │
   └── Migration Source
3. Portal status alone is not enough

Replication was validated from the SQL engine using:

sys.dm_geo_replication_link_status

This provided visibility into:

Role
Replication State
Replication Lag
Last Replication
Last Commit
4. CATCH_UP should be validated before detaching

The migration did not proceed simply because the secondary database existed.

Synchronization status was checked before the replication relationship was removed.

5. Stop Replication and Failover are different operations

This migration required an independent UAT database.

It did not require promotion of UAT as part of a disaster recovery event.

Therefore:

Stop Replication    ✔

Failover            Not required
Forced Failover     Not required
6. Backup validation is part of migration validation

The UAT database initially became:

ONLINE
READ_WRITE
MULTI_USER

before an earliest PITR restore point was visible.

The temporary production clone was retained until UAT recovery capability was confirmed.

7. Don't remove the source too early

A safer workflow is:

Replication Complete
        │
        ▼
Stop Replication
        │
        ▼
Validate UAT
        │
        ▼
Validate Backup/PITR
        │
        ▼
Application Validation
        │
        ▼
Delete Temporary Source
8. Temporary cloud resources should have an exit plan

Large databases can generate significant compute and storage costs.

Migration designs should explicitly define when temporary resources can safely be removed.

✅ Validation Checklist

Use this checklist before declaring the migration complete.

Replication
 Temporary PROD clone created
 Geo-replication configured
 Secondary database online
 Replication state = CATCH_UP
 Replication lag acceptable
 Last commit validated
Detachment
 Replication stopped
 No forced failover performed
 UAT database converted to standalone
UAT Database
 Database status = ONLINE
 Updateability = READ_WRITE
 User access = MULTI_USER
 Replication DMV returns no active link
 Application connectivity validated
 Critical data validated
Recovery
 PITR retention configured
 Earliest restore point available
 Recovery requirements reviewed
Cleanup
 UAT migration accepted
 Temporary PROD clone identified correctly
 No replication dependency remains
 Temporary PROD clone deleted
 UAT database revalidated after deletion
📚 Useful SQL Queries
Check Database State
SELECT
    DB_NAME() AS DatabaseName,
    DATABASEPROPERTYEX(DB_NAME(), 'Status') AS DatabaseStatus,
    DATABASEPROPERTYEX(DB_NAME(), 'Updateability') AS Updateability,
    DATABASEPROPERTYEX(DB_NAME(), 'UserAccess') AS UserAccess;
Check Geo-Replication Status
SELECT
    partner_server,
    partner_database,
    role_desc,
    replication_state_desc,
    replication_lag_sec,
    last_replication,
    last_commit
FROM sys.dm_geo_replication_link_status;
Confirm Replication Link Removal
SELECT *
FROM sys.dm_geo_replication_link_status;

After replication has been successfully removed, no active replication link should be returned.

🧠 Architecture Summary

The final migration pattern can be summarized as:

PROTECT
   │
   ▼
CLONE
   │
   ▼
REPLICATE
   │
   ▼
VALIDATE
   │
   ▼
DETACH
   │
   ▼
VALIDATE DATABASE
   │
   ▼
VALIDATE RECOVERY
   │
   ▼
REMOVE TEMPORARY RESOURCES

Or simply:

Clone → Replicate → Validate → Detach → Recoverability Check → Cleanup

🎯 Outcome

The migration achieved the intended objectives:

Production database remained protected.
Multi-terabyte data was made available in UAT.
PROD and UAT remained separated across subscriptions.
UAT became an independent database.
UAT was validated as read/write.
Geo-replication dependency was removed.
PITR capability was established.
Temporary production resources were removed.
Ongoing unnecessary Azure cost was eliminated.
🔒 Disclaimer

This repository represents a generalized architecture pattern and engineering case study.

It does not contain:

Real organization names
Real server names
Real database names
Real subscription IDs
Real resource group names
Real tenant IDs
Real IP addresses
Real DNS records
Credentials
Connection strings
Production screenshots
Customer information

The architecture should be adapted and tested according to the security, compliance, availability, backup, and operational requirements of each environment.

👨‍💻 Author

Arjun Singh

Cloud / Infrastructure Engineering

Areas of interest:

Microsoft Azure
Azure Architecture
Azure SQL
Cloud Migration
Infrastructure Engineering
Networking
High Availability & Disaster Recovery
Automation
DevOps
⭐ Support

If you found this architecture useful, consider giving the repository a ⭐ Star.

Feel free to fork the repository and adapt the generic architecture for your own lab or learning environment.

Tags

Azure Azure-SQL Geo-Replication Database-Migration Cloud-Architecture Microsoft-Azure DevOps Infrastructure High-Availability Disaster-Recovery PITR
