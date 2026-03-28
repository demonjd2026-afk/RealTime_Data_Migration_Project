# 🚀 On-Prem to Cloud Real-Time Data Migration Pipeline

> **End-to-end incremental data pipeline** — SQL Server → Azure Data Factory → Azure Data Lake Storage Gen2  
> Watermark-driven metadata orchestration · Star Schema DW · Parquet output · Two full implementations

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [High-Level Design (HLD)](#-high-level-design-hld)
- [Architecture Diagram](#-architecture-diagram)
- [Two Implementations](#-two-implementations)
- [Data Warehouse Design](#-data-warehouse-design)
- [Pipeline Deep Dive](#-pipeline-deep-dive)
- [Watermark Logic](#-watermark-logic)
- [Key Results](#-key-results)
- [Tech Stack](#-tech-stack)
- [Prerequisites](#-prerequisites)
- [Implementation Checklists](#-implementation-checklists)
- [Troubleshooting](#-troubleshooting)
- [Learnings & Takeaways](#-learnings--takeaways)

---

## 📖 Project Overview

Modern organizations generate large volumes of operational data in on-premise systems. To enable scalable analytics and reporting, this data must be efficiently migrated to the cloud.

This project demonstrates a **production-grade, end-to-end data engineering pipeline** that:

- Extracts data from **On-Premise SQL Server** (simulated via Azure VM) and **Azure SQL Database**
- Processes and orchestrates the load using **Azure Data Factory (ADF)**
- Stores the output in **Azure Data Lake Storage Gen2** in **Parquet format** (Snappy compressed)
- Implements a **metadata-driven incremental loading strategy** using a watermark control table — ensuring only new or updated records are processed on each run

Two separate, fully working implementations are included in this repository.

---

## 🏗 High-Level Design (HLD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HIGH-LEVEL DESIGN                                │
│                                                                         │
│   SOURCE LAYER          ORCHESTRATION LAYER         SINK LAYER          │
│                                                                         │
│  ┌─────────────┐       ┌──────────────────────┐    ┌──────────────────┐ │
│  │ SQL Server  │──────▶│  Azure Data Factory  │───▶│   ADLS Gen2      │ │
│  │ (On-Prem /  │       │                      │    │  (retaildw /     │ │
│  │  Azure SQL) │       │  ┌────────────────┐  │    │  retaildw-jay)   │ │
│  └─────────────┘       │  │ Parent Pipeline│  │    │                  │ │
│                        │  │ (Orchestration)│  │    │  Parquet files   │ │
│  ┌─────────────┐       │  └───────┬────────┘  │    │  Snappy compress │ │
│  │ Watermark   │◀──────│          │            │    │  Date-partitioned│ │
│  │ Control     │       │  ┌───────▼────────┐  │    │  TableName/      │ │
│  │ Table       │       │  │ Child Pipeline │  │    │  yyyy/MM/dd/     │ │
│  │ (ADF_       │       │  │ (Incremental)  │  │    └──────────────────┘ │
│  │  Watermark  │       │  └───────┬────────┘  │                         │
│  │  Table)     │       │          │            │                         │
│  └─────────────┘       │  ┌───────▼────────┐  │                         │
│                        │  │ Child Pipeline │  │                         │
│                        │  │ (Stored Proc)  │  │                         │
│                        │  └────────────────┘  │                         │
│                        └──────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### HLD — Component Summary

| Layer | Component | Role |
|-------|-----------|------|
| **Source** | Azure SQL DB / SQL Server on Azure VM | Operational data store (20 tables) |
| **Control** | `dbo.ADF_Watermark_Table` | Tracks last processed key per table |
| **Orchestration** | ADF — Parent Pipeline | Reads watermark table, loops over all tables |
| **Load** | ADF — Child Pipeline Incremental | Checks for new data, executes copy |
| **Control Update** | ADF — Child Pipeline StoreProcedure | Updates watermark after successful copy |
| **Sink** | ADLS Gen2 (Parquet / Snappy) | Cloud data lake — date-partitioned storage |
| **Integration** | AutoResolveIR / SHIR-VM-SQLServer | Connectivity layer between ADF and source |

---

## 🔷 Architecture Diagram

### 3-Layer Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — ORCHESTRATION : Parent Pipeline                                   │
│                                                                              │
│   ┌─────────────────────────┐        ┌──────────────────────────────────┐   │
│   │  LOOKUP                 │        │  FOREACH                         │   │
│   │  LKUP_SQLTABLE_         │──────▶ │  ForEachSQLTable                 │   │
│   │  WATRMRK_DW             │        │  @activity('LKUP_SQLTABLE_       │   │
│   │                         │        │  WATRMRK_DW').output.value       │   │
│   │  SELECT TableSchema,    │        │  (Sequential = OFF → parallel)   │   │
│   │  TableName,             │        └────────────────┬─────────────────┘   │
│   │  WatermarkColumn,       │                         │ @item() per table   │
│   │  LastLoadValue          │                         ▼                     │
│   │  FROM ADF_Watermark_    │        ┌──────────────────────────────────┐   │
│   │  Table                  │        │  EXECUTE PIPELINE                │   │
│   │  First row only: OFF    │        │  EXEC_SQL_INCREMENTAL_PIPELINE   │   │
│   └─────────────────────────┘        │  → calls Layer 2                 │   │
│                                      └──────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — LOAD : Child Pipeline Incremental                                 │
│                                                                              │
│  ┌──────────────┐    ┌─────────────────────┐    ┌──────────────────────┐    │
│  │  LOOKUP      │    │  IF CONDITION        │    │  COPY DATA           │    │
│  │  LKUP_MAX_   │──▶ │  IF_NEW_RECORD_      │──▶ │  COPYSQL_FULL_       │    │
│  │  VALUE       │    │  EXISTS              │    │  INCTLOAD_TABLES     │    │
│  │              │    │                      │    │                      │    │
│  │  SELECT MAX  │    │  TRUE if:            │    │  Source: SQL DB      │    │
│  │  (Watermark  │    │  LastLoadValue=NULL  │    │  SELECT * FROM table │    │
│  │  Column)     │    │  OR MAX > LastLoad   │    │  WHERE col > lastVal │    │
│  │  AS MaxValue │    │                      │    │  (NULL = full load)  │    │
│  │              │    │  FALSE → skip table  │    │                      │    │
│  └──────────────┘    └─────────────────────┘    │  Sink: ADLS Gen2     │    │
│                                                  │  Format: Parquet     │    │
│                                                  │  Compress: Snappy    │    │
│                                                  └──────────┬───────────┘    │
│                                                             │ on success     │
│                                                             ▼                │
│                                                  ┌──────────────────────┐    │
│                                                  │  EXECUTE PIPELINE    │    │
│                                                  │  EXEC_UPDATE_        │    │
│                                                  │  WATERMARK           │    │
│                                                  │  → calls Layer 3     │    │
│                                                  └──────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — CONTROL : Child Pipeline StoreProcedure                           │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  STORED PROCEDURE ACTIVITY                                           │  │
│   │  SP-UPDT_WTRMRK → [dbo].[UpdateWatermark]                           │  │
│   │                                                                      │  │
│   │  Dynamically computes MAX(WatermarkColumn) from source table         │  │
│   │  Updates ADF_Watermark_Table SET LastLoadValue = MAX value           │  │
│   │  WHERE TableName = @TableName AND TableSchema = @TableSchema         │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔀 Two Implementations

This repository covers two fully working implementations of the same pipeline:

### Implementation 1 — Azure SQL DB → ADF → ADLS Gen2

| Field | Value |
|-------|-------|
| **Data Source** | Azure SQL Database (cloud-managed PaaS) |
| **Integration Runtime** | AutoResolveIntegrationRuntime (built-in, no install) |
| **SQL Linked Service** | `LS_AzureSQL_DW` |
| **Source Dataset** | `DS_LKUP_MAX_VALUE` |
| **Sink Container** | `retaildw` |
| **SHIR Required** | ❌ No |
| **Setup Time** | ~2.5 hours |

**Key advantage:** Purely cloud-native. No Self-Hosted IR needed. ADF connects directly to Azure SQL DB via AutoResolveIntegrationRuntime. Your laptop can be switched off after setup.

---

### Implementation 2 — On-Prem SQL Server via Azure VM → ADF → ADLS Gen2

| Field | Value |
|-------|-------|
| **Data Source** | SQL Server Express on Azure VM (IaaS — simulates on-premises) |
| **Integration Runtime** | `SHIR-VM-SQLServer` (Self-Hosted IR installed on VM) |
| **SQL Linked Service** | `LS_VM_SQL_DW` (connects via private IP e.g. `10.0.0.4,1433`) |
| **Source Dataset** | `DS_LKUP_MAX_VALUE_VM` |
| **Sink Container** | `retaildw-jay` |
| **SHIR Required** | ✅ Yes — installed on VM |
| **Java JRE Required** | ✅ Yes — required by SHIR for Parquet serialization |
| **VM Size** | Standard_D2s_v3 (2 vCPUs, 8 GiB) ~$137/month |
| **Setup Time** | ~2.5 hours |

**Key advantage:** Mirrors a real enterprise on-premises migration. SHIR bridges ADF to the on-VM SQL Server over Azure's private network. No Mac dependency — laptop can be off once the VM is running.

---

## 🗄 Data Warehouse Design

Star Schema optimized for analytics and BI queries.

### Dimension Tables (10)

| Table | Primary Key | Row Count |
|-------|-------------|-----------|
| DimCustomer | CustomerKey | 10,000 |
| DimProduct | ProductKey | 1,000 |
| DimStore | StoreKey | 50 |
| DimEmployee | EmployeeKey | ~160 |
| DimPromotion | PromotionKey | 4 |
| DimRegion | RegionKey | 5 |
| DimCategory | CategoryKey | 5 |
| DimPaymentMethod | PaymentMethodKey | 5 |
| DimSupplier | SupplierKey | 100 |
| DimDate | DateKey | 20,241,231 |

### Fact Tables (10)

| Table | Primary Key | Row Count |
|-------|-------------|-----------|
| FactSales | SalesKey | 500,000 |
| FactOrders | OrderKey | ~25,000 |
| FactPayments | PaymentKey | ~25,000 |
| FactInventory | InventoryKey | ~25,000 |
| FactReturns | ReturnKey | ~25,000 |
| FactShipment | ShipmentKey | ~25,000 |
| FactStoreSales | StoreSalesKey | ~25,000 |
| FactCustomerActivity | ActivityKey | ~25,000 |
| FactProductPerformance | PerformanceKey | 100,000 |
| FactDiscounts | DiscountKey | ~25,000 |

---

## 🔬 Pipeline Deep Dive

### ADF Components

#### Linked Services

| Name | Type | IR | Purpose |
|------|------|----|---------|
| `LS_AzureSQL_DW` | Azure SQL Database | AutoResolveIR | Impl 1 — Azure SQL source |
| `LS_VM_SQL_DW` | SQL Server | SHIR-VM-SQLServer | Impl 2 — VM SQL Server source |
| `ADLS_SQL_PARQUET` | ADLS Gen2 | AutoResolveIR | Impl 1 — ADLS sink |
| `ADLS_SQL_VM_PARQUET` | ADLS Gen2 | AutoResolveIR | Impl 2 — ADLS sink |

#### Datasets

| Name | Type | Parameters | Purpose |
|------|------|------------|---------|
| `DS_LKUP_MAX_VALUE` | Azure SQL DB | `schemaname`, `tablename` | Impl 1 source dataset |
| `DS_LKUP_MAX_VALUE_VM` | SQL Server | `schemaname`, `tablename` | Impl 2 source dataset |
| `DS_SqlTable_Parquet1` | ADLS Gen2 Parquet | `Table_Name` | Sink dataset — both impls |

#### Pipelines

| Pipeline | Parameters | Activities |
|----------|------------|------------|
| Parent Pipeline Orchestration | — | Lookup, ForEach, Execute Pipeline |
| Child Pipeline Incremental | TableSchema, TableName, WatermarkColumn, LastLoadValue | Lookup, IF Condition, Copy Data, Execute Pipeline |
| Child Pipeline StoreProcedure | TableSchema, TableName, WatermarkColumn | Stored Procedure |

### Key ADF Expressions

**Lookup query — MAX watermark value:**
```sql
SELECT MAX(@{pipeline().parameters.WatermarkColumn}) AS MaxValue
FROM @{pipeline().parameters.TableSchema}.@{pipeline().parameters.TableName}
```

**IF Condition — check for new records:**
```
@or(
  equals(pipeline().parameters.LastLoadValue, null),
  greater(
    int(activity('LKUP_MAX_VALUE').output.firstRow.MaxValue),
    int(pipeline().parameters.LastLoadValue)
  )
)
```

**Copy Activity — dynamic source query (full / incremental):**
```
@concat(
  'SELECT * FROM ',
  pipeline().parameters.TableSchema, '.', pipeline().parameters.TableName,
  if(
    equals(pipeline().parameters.LastLoadValue, null), '',
    concat(
      ' WHERE ', pipeline().parameters.WatermarkColumn,
      ' > ''', pipeline().parameters.LastLoadValue, ''''
    )
  )
)
```

**Sink — folder path expression:**
```
@concat(dataset().Table_Name,'/',formatDateTime(utcNow(),'yyyy/MM/dd'))
```

**Sink — file name expression:**
```
@concat(dataset().Table_Name,'_',formatDateTime(utcNow(),'yyyyMMddHHmmss'),'.parquet')
```

---

## 💧 Watermark Logic

```
FIRST RUN (LastLoadValue = 0 or NULL)
    └──▶ IF Condition = TRUE
         └──▶ SELECT * FROM table  (full load — no WHERE clause)
              └──▶ Copy all rows to ADLS
                   └──▶ UpdateWatermark → LastLoadValue = MAX(PrimaryKey)

SUBSEQUENT RUNS (LastLoadValue = N)
    └──▶ IF Condition: MAX(PrimaryKey) > N ?
         ├── TRUE  → SELECT * FROM table WHERE PrimaryKey > N  (incremental)
         │          └──▶ Copy only new rows to ADLS
         │               └──▶ UpdateWatermark → LastLoadValue = new MAX
         └── FALSE → Skip table entirely (no copy, no SP call)
```

### Watermark Control Table Schema

```sql
CREATE TABLE dbo.ADF_Watermark_Table (
    TableSchema      VARCHAR(50)  NULL,
    TableName        VARCHAR(200) NULL,
    WatermarkColumn  VARCHAR(100) NULL,
    LastLoadValue    BIGINT       NULL   -- 0 = trigger full load on first run
);
```

### UpdateWatermark Stored Procedure

```sql
CREATE OR ALTER PROCEDURE [dbo].[UpdateWatermark]
    @TableSchema     VARCHAR(50),
    @TableName       VARCHAR(100),
    @WatermarkColumn VARCHAR(100)
AS BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @MaxValue BIGINT;
    SET @SQL = 'SELECT @MaxVal = MAX(' + QUOTENAME(@WatermarkColumn) + ')
                FROM ' + QUOTENAME(@TableSchema) + '.' + QUOTENAME(@TableName);
    EXEC sp_executesql @SQL, N'@MaxVal BIGINT OUTPUT', @MaxVal = @MaxValue OUTPUT;
    IF @MaxValue IS NOT NULL
        UPDATE dbo.ADF_Watermark_Table
        SET LastLoadValue = @MaxValue
        WHERE TableName = @TableName AND TableSchema = @TableSchema;
END
```

---

## ✅ Key Results

| Metric | Value |
|--------|-------|
| Total tables migrated | 20 (10 Dim + 10 Fact) |
| FactSales rows migrated | 500,000 |
| Full pipeline run time | ~22 minutes |
| Incremental run time | ~9 minutes (60% faster) |
| Output format | Parquet + Snappy compression |
| Folder structure | `TableName/yyyy/MM/dd/TableName_yyyyMMddHHmmss.parquet` |
| Incremental accuracy | Only new rows copied on re-run ✅ |
| Watermark updated | Correctly after every pipeline run ✅ |

---

## 🛠 Tech Stack

| Tool / Service | Purpose |
|----------------|---------|
| Azure Data Factory | Pipeline orchestration, data movement |
| Azure SQL Database | Cloud SQL source (Implementation 1) |
| SQL Server Express on Azure VM | On-prem SQL source simulation (Implementation 2) |
| Azure Data Lake Storage Gen2 | Cloud data lake sink |
| Self-Hosted Integration Runtime (SHIR) | ADF ↔ On-prem VM connectivity |
| AutoResolveIntegrationRuntime | ADF ↔ Azure SQL connectivity (cloud-native) |
| DBeaver | SQL client for DB management & scripting |
| T-SQL | DDL, DML, Dynamic SQL, Stored Procedures |
| Parquet + Snappy | Columnar storage format with compression |
| Java JRE | Required by SHIR for Parquet file serialization |
| Azure Portal | VM, NSG, Storage account, ADF provisioning |

---

## 📋 Prerequisites

### For Implementation 1 (Azure SQL DB)
- Active Azure subscription
- Azure SQL Server & Database provisioned
- Azure Data Factory instance
- Azure Data Lake Storage Gen2 account with a container (`retaildw`)
- DBeaver (or any SQL client)

### For Implementation 2 (Azure VM)
All of the above, plus:
- Azure Virtual Machine — Windows Server 2022, Standard_D2s_v3
- NSG inbound rule: TCP port 1433 (Allow-SQL-ADF)
- SQL Server Express installed on VM
- TCP/IP enabled on port 1433 in SQL Server Configuration Manager
- Windows Firewall inbound rule for port 1433
- SQL Server Authentication enabled + `sqladmin` login created
- SHIR installed and registered on the VM
- Java JRE installed on VM + `JAVA_HOME` environment variable set
- SHIR service restarted after Java installation

---

## ✔ Implementation Checklists

<details>
<summary><strong>Implementation 1 — Azure SQL DB → ADF → ADLS Gen2</strong></summary>

**Phase 1 — Database Setup**
- [ ] DBeaver connected to Azure SQL Server
- [ ] All 10 Dimension tables created and seeded
- [ ] All 10 Fact tables created and seeded
- [ ] `dbo.ADF_Watermark_Table` created — all 20 rows, LastLoadValue = 0
- [ ] `dbo.UpdateWatermark` stored procedure created

**Phase 2 — Linked Services & Datasets**
- [ ] Azure SQL firewall: Allow Azure services = ON
- [ ] `LS_AzureSQL_DW` — Test connection green
- [ ] `ADLS_SQL_PARQUET` — Test connection green
- [ ] `DS_LKUP_MAX_VALUE` — Parameters: schemaname + tablename; Connection tab expressions set
- [ ] `DS_SqlTable_Parquet1` — Parameter: Table_Name; Folder path + File name expressions set

**Phase 3 — SP Pipeline**
- [ ] Child Pipeline StoreProcedure — 3 parameters defined
- [ ] SP activity → `[dbo].[UpdateWatermark]` — all 3 params mapped
- [ ] Validate → 0 errors

**Phase 4 — Child Incremental Pipeline**
- [ ] 4 parameters defined (TableSchema, TableName, WatermarkColumn, LastLoadValue)
- [ ] Lookup → First row only = ON; Query = SELECT MAX(...)
- [ ] IF Condition expression set correctly
- [ ] Copy source query — dynamic SELECT with full/incremental logic
- [ ] Copy sink → DS_SqlTable_Parquet1 with Table_Name mapped
- [ ] Validate → 0 errors

**Phase 5 — Parent Pipeline**
- [ ] Lookup → First row only = OFF (returns all rows)
- [ ] ForEach → Items = @activity(...).output.value
- [ ] Execute Pipeline → all 4 @item() parameters mapped
- [ ] Validate → 0 errors; Publish all

**Phase 6 — Testing**
- [ ] Debug run — all 22 activities Succeeded
- [ ] Parquet files visible in ADLS under correct folder structure
- [ ] Watermark table — no zeros remain
- [ ] Incremental test — only new rows in new Parquet file
</details>

<details>
<summary><strong>Implementation 2 — On-Prem via Azure VM → ADF → ADLS Gen2</strong></summary>

**VM Setup**
- [ ] VM created — Windows Server 2022, Standard_D2s_v3
- [ ] NSG inbound rule: TCP 1433 Allow (Allow-SQL-ADF)
- [ ] SQL Server Express installed; TCP/IP enabled on port 1433
- [ ] Windows Firewall inbound rule for port 1433
- [ ] SQL Server Authentication enabled; sqladmin login created
- [ ] Java JRE installed; JAVA_HOME set; %JAVA_HOME%\bin added to PATH
- [ ] SHIR installed and registered → Status: Connected to cloud service
- [ ] SHIR service restarted after Java installation

**Database Setup (same as Impl 1 — run on VM)**
- [ ] All 20 tables + watermark table + UpdateWatermark SP created on VM

**ADF Updates**
- [ ] SHIR-VM-SQLServer IR created in ADF
- [ ] VM private IP noted (e.g. 10.0.0.4)
- [ ] `LS_VM_SQL_DW` — SQL Server type; SHIR; private IP; Test connection green
- [ ] `DS_LKUP_MAX_VALUE_VM` created with SHIR and parameters
- [ ] `ADLS_SQL_VM_PARQUET` linked service created
- [ ] `DS_SqlTable_Parquet1` updated to use `ADLS_SQL_VM_PARQUET`; retaildw-jay container
- [ ] SP activity updated → LS_VM_SQL_DW + SHIR
- [ ] Lookup updated → DS_LKUP_MAX_VALUE_VM + SHIR
- [ ] IF condition updated to reference `LKUP_MAX_VALUE_VM`
- [ ] Copy source → DS_LKUP_MAX_VALUE_VM
- [ ] Publish all; Validate → 0 errors

**Testing**
- [ ] Full load — all 22 activities Succeeded (~22 min)
- [ ] All 20 Parquet folders in retaildw-jay
- [ ] Watermark — all 20 LastLoadValues populated (FactSales = 500,000)
- [ ] 5 new rows inserted into FactSales
- [ ] Incremental re-run — Succeeded (~9 min); only FactSales copied
- [ ] FactSales LastLoadValue = 500,005
- [ ] New 1.69 KiB Parquet file in ADLS
</details>

---

## 🔧 Troubleshooting

| Error / Symptom | Likely Cause | Fix |
|----------------|--------------|-----|
| Test connection fails (ADF) | NSG blocking port 1433 | Portal → VM → Networking → verify Allow-SQL-ADF rule: TCP 1433, Allow |
| Test connection fails (DBeaver) | Windows Firewall blocking | RDP → Windows Defender Firewall → Inbound Rules → TCP 1433, Allow |
| Login failed for sqladmin | SQL Auth not enabled | SSMS → Server Properties → Security → SQL + Windows Auth → restart service |
| Pipeline times out | VM stopped / deallocated | Portal → VM → Start. Wait 2 min, then re-run |
| TCP/IP not available | TCP/IP disabled in SQL config | SQL Server Configuration Manager → TCP/IP → Enable → restart service |
| SHIR shows Offline in ADF | SHIR service stopped on VM | RDP → IRTCM → Start Service. Or: `net start DIAHostService` |
| JreNotFound / jvm.dll error | Java not installed on VM | Install from java.com, set JAVA_HOME, add to PATH, restart SHIR |
| Linked service mismatch error | Used Azure SQL Database type instead of SQL Server | Delete LS and recreate as **SQL Server** connector type |
| 0 rows copied to ADLS | LastLoadValue = current MAX | `UPDATE dbo.ADF_Watermark_Table SET LastLoadValue = 0` for fresh full load |
| Publish all greyed out | Validation errors present | Click **Validate all** first and fix all errors shown |
| Schema import failed error | @dataset() expression added during Set Properties | Leave Table name blank during Set Properties; add expression only in Connection tab |

---

## 💡 Learnings & Takeaways

- **Metadata-driven design** — a single watermark control table drives the entire pipeline dynamically, eliminating hardcoded table references
- **Dynamic SQL patterns** — using `@concat` with `if()` to toggle between full and incremental load in a single Copy Activity
- **SHIR vs AutoResolveIR** — understanding when and why a Self-Hosted IR is needed vs the built-in cloud IR
- **Java dependency on SHIR** — SHIR uses Apache Parquet Java libraries for Parquet serialization; Java JRE must be installed and `JAVA_HOME` set on the SHIR host machine
- **Dataset parameterization** — using `@dataset().paramname` expressions to make a single dataset serve 20 different tables
- **ForEach parallel processing** — unchecking Sequential enables ADF to process multiple tables concurrently, significantly reducing total run time
- **Cost control** — Azure VM at $137/month running 24/7; always deallocate (`az vm deallocate`) when not in use; only disk cost (~$1–2/month) applies when deallocated
- **Parquet + Snappy** — columnar format reduces storage footprint and speeds up downstream analytics queries significantly vs row-based formats

---

## 📄 License

This project is for educational and portfolio purposes.

---

*Built with ❤️ as part of the VisionBoard Data Engineering Program — Batch 10*