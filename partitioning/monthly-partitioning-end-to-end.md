# SQL Server Monthly Partitioning – End-to-End Operational Guide

This document describes a **production-ready partitioning workflow** in SQL Server, covering:
- Initial monthly partitioning
- Proactive future partition maintenance
- Late partition recovery using **Pattern 3 (Historical vs Live data separation)**

---

## Assumptions

- SQL Server (Enterprise recommended)
- Table: `dbo.Sales`
- Partition key: `SalesDate` (DATE)
- Partition granularity: **Monthly**
- Partition type: `RANGE RIGHT`
- Each month uses **one FILEGROUP + one DATA FILE**

---

# STEP 1 — Initial Partitioning for Year 2024

## 1.1 Create Filegroups and Data Files for 2024

Example for January 2024 (repeat for each month):

```sql
ALTER DATABASE MyDB ADD FILEGROUP FG_2024_01;

ALTER DATABASE MyDB ADD FILE
(
    NAME = 'Sales_2024_01',
    FILENAME = 'D:\SQLData\Sales_2024_01.ndf',
    SIZE = 50GB,
    FILEGROWTH = 5GB
)
TO FILEGROUP FG_2024_01;
```
FG_2024_01 … FG_2024_12
FG_FUTURE       -- catch-all for future data

## 1.2 Create Filegroups and Data Files for 2024
```sql
CREATE PARTITION FUNCTION pf_SalesDate (date)
AS RANGE RIGHT FOR VALUES
(
    '2024-01-01','2024-02-01','2024-03-01','2024-04-01',
    '2024-05-01','2024-06-01','2024-07-01','2024-08-01',
    '2024-09-01','2024-10-01','2024-11-01','2024-12-01'
);
```

## 1.3 Create Partition Scheme
```sql
CREATE PARTITION SCHEME ps_SalesDate
AS PARTITION pf_SalesDate
TO
(
    FG_2024_01, FG_2024_02, FG_2024_03, FG_2024_04,
    FG_2024_05, FG_2024_06, FG_2024_07, FG_2024_08,
    FG_2024_09, FG_2024_10, FG_2024_11, FG_2024_12,
    FG_FUTURE
);
```

## 1.4 Create Partition Table dbo.Sales
```sql
CREATE TABLE dbo.Sales
(
    SalesID     BIGINT IDENTITY(1,1) NOT NULL,
    SalesDate   DATE   NOT NULL,
    Amount      MONEY  NOT NULL,
    CustomerID  INT    NOT NULL,
    CONSTRAINT PK_Sales
        PRIMARY KEY CLUSTERED (SalesDate, SalesID)
        ON ps_SalesDate (SalesDate)
);
```
---

# STEP 2 — Proactively Add Partitions for 2025 (End of 2024)
Best pratice: **Always add future partitions before data arrives**

## 2.1 Create Filegroup and Data Files for 2025
Example for January 2025:
``` sql
ALTER DATABASE MyDB ADD FILEGROUP FG_2025_01;

ALTER DATABASE MyDB ADD FILE
(
    NAME = 'Sales_2025_01',
    FILENAME = 'D:\SQLData\Sales_2025_01.ndf',
    SIZE = 50GB,
    FILEGROWTH = 5GB
)
TO FILEGROUP FG_2025_01;
```
Repeat for FG_2025_01 ... FG_2025_12

## 2.2 Add Partitions (Metadata-only)
```sql
ALTER PARTITION SCHEME ps_SalesDate NEXT USED FG_2025_01;
ALTER PARTITION FUNCTION pf_SalesDate() SPLIT RANGE ('2025-01-01');

ALTER PARTITION SCHEME ps_SalesDate NEXT USED FG_2025_02;
ALTER PARTITION FUNCTION pf_SalesDate() SPLIT RANGE ('2025-02-01');

-- repeat until 2025-12-01
```
Result:
- No data movement
- No locking impact
- Future data routed correctly

---

# STEP 3 — Late Partition Recovery (Mid-2025) Using "BOUNDARY FREEZE + DELTA CATCHUP
Best pratice: **Always add future partitions before data arrives**
**Scenario**
- It is July 2025
- Partitions for 2025 were not created
- Data from 2025-01-01 onward resides in FG_FUTURE
- System is still receiving live data

## 3.1 Define Cutoff Date
```sql
- CutoffDate = '2025-07-01'
```
Meaning:
- < CutoffDate → historical data (safe to process)
- >= CutoffDate → live data (must not be touched)

## 3.2 Create Historical Staging table
```sql
CREATE TABLE dbo.Sales_Stage_2025_H1
(
    -- schema must match dbo.Sales exactly
)
ON FG_STAGE;
```
Add CHECK constraint (required for SWITCH):
```sql
ALTER TABLE dbo.Sales_Stage_2025_H1
ADD CONSTRAINT CK_Sales_2025_H1
CHECK
(
    SalesDate >= '2025-01-01'
    AND SalesDate <  '2025-07-01'
);
```

## 3.3 Copy Historical data (Controlled IO)
``` sql
INSERT INTO dbo.Sales_Stage_2025_H1
SELECT *
FROM dbo.Sales
WHERE SalesDate >= '2025-01-01'
  AND SalesDate <  '2025-07-01';
```
Live data (>= 2025-07-01) remains untouched.

## 3.4 Delete Historical Data from FUTURE Partition (Batched)
``` sql
DECLARE @d date = '2025-01-01';

WHILE @d < '2025-07-01'
BEGIN
    DELETE FROM dbo.Sales
    WHERE SalesDate >= @d
      AND SalesDate < DATEADD(day, 1, @d);

    CHECKPOINT;
    SET @d = DATEADD(day, 1, @d);
END
```
Result:
- FUTURE partition contains only live data
- No race condition with inserts

## 3.5 Add Missing Partitions for 2025 (Metadata-only)
```sql
ALTER PARTITION SCHEME ps_SalesDate NEXT USED FG_2025_01;
ALTER PARTITION FUNCTION pf_SalesDate() SPLIT RANGE ('2025-01-01');

-- repeat for each month up to 2025-06-01
```
No data movement occurs because historical rows were already removed.

## 3.6 Switch Historical Data into Correct Partitions
``` sql
ALTER TABLE dbo.Sales_Stage_2025_H1
SWITCH TO dbo.Sales PARTITION <partition_number_for_2025_H1>;
```
This operation is metadata-only:
- No data movement
- No logging
- No IO

## 3.7 Clean up
``` sql
DROP TABLE dbo.Sales_Stage_2025_H1;
```
---

## Key Takeaways
- Always add partitions early whenever possible
- SPLIT only moves data if rows exist in the new range
- SWITCH is metadata-only but requires exact structural alignment
- Pattern 3 safely fixes partitioning mistakes without touching live data
- Large deletes are acceptable when batched and controlled




