#  Enterprise ODS ETL Pipeline — SSIS + SQL Server

> An end-to-end **Operational Data Store (ODS)** built with SQL Server Integration Services (SSIS), consolidating data from multiple source systems into a unified, query-ready data store.

---

##  Project Overview

This project implements a production-style ODS that ingests data from two transactional source systems (**Sales** and **HR**), applies data quality rules, and loads clean, integrated data into a centralized store for operational reporting and analytics.

---

##  Architecture

```
SourceDB_Sales          SourceDB_HR
   (Customers,            (Employees,
   Products,              Departments,
   SalesOrders,           Attendance,
   OrderDetails)          Payroll)
         │                     │
         └──────────┬──────────┘
                    ▼
           ┌─────────────────┐
           │  Staging Layer  │  ← stg schema (raw, unvalidated)
           └────────┬────────┘
                    ▼
           ┌─────────────────┐
           │Integration Layer│  ← int schema (cleansed, transformed)
           └────────┬────────┘
                    ▼
           ┌─────────────────┐
           │  Metadata Layer │  ← meta schema (ETL logs, load control)
           └─────────────────┘
```

---

##  Tech Stack

| Tool | Purpose |
|------|---------|
| SQL Server 2019 | Database Engine |
| SSIS (SQL Server Integration Services) | ETL Orchestration |
| SSMS | Database Management |
| SSDT / Visual Studio | SSIS Package Development |
| T-SQL | Data transformation logic |

---

##  SSIS Packages

| Package | Type | Load Strategy | Lookup |
|---------|------|---------------|--------|
| `Load_Departments.dtsx` | Dimension | Full Load | — |
| `Load_Customers.dtsx` | Dimension | Incremental (SCD Type 1) | — |
| `Load_Products.dtsx` | Dimension | Incremental (SCD Type 1) | — |
| `Load_Employees.dtsx` | Dimension | Incremental (SCD Type 1) | `DepartmentKey` |
| `Load_SalesOrders.dtsx` | Fact | Incremental | `CustomerKey` |
| `Load_OrderDetails.dtsx` | Fact | Insert-Only | `OrderKey`, `ProductKey` |
| `Load_Attendance.dtsx` | Fact | Insert-Only | `EmployeeKey` |
| `Load_Payroll.dtsx` | Fact | Insert-Only | `EmployeeKey` |
| `Master_ODS.dtsx` | Orchestration | Runs all packages in sequence | — |

---

##  ETL Design Patterns Used

- **Incremental Load** — extracts only modified records using `LastLoadDate` watermarking
- **SCD Type 1** — overwrites changed dimension attributes in-place
- **Lookup Transformation** — resolves foreign keys (e.g. `DepartmentKey`, `CustomerKey`) before loading facts
- **Staging → Integration pattern** — data lands raw in `stg`, then merged/cleansed into `int`
- **Insert-Only** — for append-only fact tables using `NOT EXISTS` guard

---

##  Repository Structure

```
ODS-ETL-Project/
│
├── SQL/
│   ├── 01_Source_Databases_Setup.sql      # Creates SourceDB_Sales & SourceDB_HR
│   └── 02_ODS_Database_Setup.sql          # Creates ODS_Enterprise + all schemas/tables
│
├── SSIS/
│   └── ODS_ETL_Project/
│       ├── Load_Customers.dtsx
│       ├── Load_Products.dtsx
│       ├── Load_Departments.dtsx
│       ├── Load_Employees.dtsx
│       ├── Load_SalesOrders.dtsx
│       ├── Load_OrderDetails.dtsx
│       ├── Load_Attendance.dtsx
│       ├── Load_Payroll.dtsx
│       └── Master_ODS.dtsx
│
├── docs/
│   └── architecture.png
│
└── README.md
```

---

##  Getting Started

### Prerequisites
- SQL Server 2019 or 2022 (with SSIS feature installed)
- SQL Server Management Studio (SSMS)
- Visual Studio with SSIS extension (SSDT)

### Setup Steps

**1. Create Source Databases**
```sql
-- Run in SSMS
01_Source_Databases_Setup.sql
```

**2. Create ODS Database**
```sql
02_ODS_Database_Setup.sql
```

**3. Open SSIS Project**
- Open `SSIS/ODS_ETL_Project/` in Visual Studio
- Update Connection Managers to point to your SQL Server instance

**4. Run Master Package**
- Right-click `Master_ODS.dtsx` → Execute Package
- Monitor execution in the Progress tab

---

##  Validation

```sql
-- Check row counts across all tables
SELECT 'Customers'    AS TableName, COUNT(*) AS RowCount FROM [int].Customers
UNION ALL SELECT 'Products',    COUNT(*) FROM [int].Products
UNION ALL SELECT 'Departments', COUNT(*) FROM [int].Departments
UNION ALL SELECT 'Employees',   COUNT(*) FROM [int].Employees
UNION ALL SELECT 'SalesOrders', COUNT(*) FROM [int].SalesOrders
UNION ALL SELECT 'OrderDetails',COUNT(*) FROM [int].OrderDetails
UNION ALL SELECT 'Attendance',  COUNT(*) FROM [int].Attendance
UNION ALL SELECT 'Payroll',     COUNT(*) FROM [int].Payroll;

-- Check ETL execution log
SELECT PackageName, Status, RowsExtracted, RowsLoaded, ExecutionStartTime
FROM meta.ETL_ExecutionLog
ORDER BY ExecutionStartTime DESC;

-- Verify load timestamps
SELECT * FROM meta.TableLoadControl
ORDER BY LastSuccessfulLoadDate DESC;

-- Incremental test: update source record, re-run package, verify only changed row updated
UPDATE SourceDB_Sales.dbo.Customers SET Phone = '555-9999' WHERE CustomerID = 1;
```

---

##  Key Features

- ✅ **Full orchestration** via Master Package — dimensions always load before facts
- ✅ **ETL metadata logging** — every execution tracked in `meta` schema
- ✅ **Error handling** — event handlers configured on all packages
- ✅ **Incremental loading** — minimizes load time on subsequent runs
- ✅ **Multi-source integration** — Sales and HR systems unified in one ODS

---

[LinkedIn URL] | [Email]
