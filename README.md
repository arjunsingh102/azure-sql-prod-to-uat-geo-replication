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


