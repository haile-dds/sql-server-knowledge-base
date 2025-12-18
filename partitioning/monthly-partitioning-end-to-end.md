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

Repeat for FG_2025_01 ... FG_2025_12
```

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
```
- CutoffDate = '2025-07-01'
```
Meaning:
- < CutoffDate → historical data (safe to process)
- >= CutoffDate → live data (must not be touched)

## 3.2 Create Historical Staging tables
Example for January 2025:
```sql
CREATE TABLE dbo.Sales_Stage_2025_01
(
    -- schema must match dbo.Sales exactly
)
ON FG_STAGE;
```
Add CHECK constraint (required for SWITCH):
```sql
ALTER TABLE dbo.Sales_Stage_2025_01
ADD CONSTRAINT CK_Sales_2025_01
CHECK
(
    SalesDate >= '2025-01-01'
    AND SalesDate <  '2025-02-01'
);

Repeat for 5 remaining tables: Sales_Stage_2025_02 - Sales_Stage_2025_06
```

## 3.3 Copy Historical data (Controlled IO)
Example for January 2025:
``` sql
INSERT INTO dbo.Sales_Stage_2025_01
SELECT *
FROM dbo.Sales
WHERE SalesDate >= '2025-01-01'
  AND SalesDate <  '2025-02-01';

Repeat for 5 remaining tables: Sales_Stage_2025_02 - Sales_Stage_2025_06
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
ALTER TABLE dbo.Sales_Stage_2025_01
SWITCH TO dbo.Sales PARTITION <partition_number_for_2025_01>;

repeat for each month up to 2025-06-01
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

---

# Post-Partition SPLIT / SWITCH Checklist

This checklist should be executed **after any partition SPLIT or SWITCH operation** to ensure
correctness, performance, and operational safety.

---

## 1. Verify Partition Boundaries

Ensure the partition function contains the expected boundary values.

```sql
SELECT
    pf.name AS PartitionFunction,
    prv.boundary_id,
    prv.value AS BoundaryValue
FROM sys.partition_functions pf
JOIN sys.partition_range_values prv
    ON pf.function_id = prv.function_id
WHERE pf.name = 'pf_SalesDate'
ORDER BY prv.boundary_id;
```
✔ Expected:
- All month boundaries exist
- Boundaries are in correct chronological order

---

## 2. Verify Partition → Filegroup Mapping
Confirm that each partition maps to the intended filegroup.
```sql
SELECT
    ps.name AS PartitionScheme,
    dds.destination_id AS PartitionNumber,
    fg.name AS FilegroupName
FROM sys.partition_schemes ps
JOIN sys.destination_data_spaces dds
    ON ps.data_space_id = dds.partition_scheme_id
JOIN sys.filegroups fg
    ON dds.data_space_id = fg.data_space_id
WHERE ps.name = 'ps_SalesDate'
ORDER BY dds.destination_id;
```
✔ Expected:
- Each month partition maps to its dedicated filegroup
- No partition accidentally mapped to the wrong filegroup

---

## 3. Verify Row Distribution Per Partition
Validate that rows are distributed correctly across partitions.
```sql
SELECT
    p.partition_number,
    p.rows
FROM sys.partitions p
WHERE p.object_id = OBJECT_ID('dbo.Sales')
  AND p.index_id IN (0,1)
ORDER BY p.partition_number;
```
✔ Expected:
- Historical partitions contain expected row counts
- FUTURE / current partition contains only live data

---

## 4. Verify Index Partition Alignment
Ensure indexes are partition-aligned with the base table.
```sql
SELECT
    i.name AS IndexName,
    i.index_id,
    ds.name AS DataSpace,
    ps.name AS PartitionScheme
FROM sys.indexes i
JOIN sys.data_spaces ds
    ON i.data_space_id = ds.data_space_id
LEFT JOIN sys.partition_schemes ps
    ON ds.data_space_id = ps.data_space_id
WHERE i.object_id = OBJECT_ID('dbo.Sales')
ORDER BY i.index_id;
```
✔ Expected:
- Partitioned indexes use the same partition scheme as the clustered index
- Misaligned indexes are explicitly justified

---

## 5. Check for Fragmentation (Only if Data Movement Occurred)
Only required if:
- SPLIT was executed late
- Data was moved between partitions
```sql
SELECT
    ips.partition_number,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats
(
    DB_ID(),
    OBJECT_ID('dbo.Sales'),
    1,      -- clustered index
    NULL,
    'SAMPLED'
) ips
WHERE ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;
```
Guidance:
- < 10% → no action
- 10–30% → consider REORGANIZE
- > 30% → consider REBUILD (partition-level)

---

## 6. Rebuild Indexes (Partition-Level Only, If Needed)
⚠️ Do NOT rebuild the entire table unless absolutely required.
```sql
ALTER INDEX PK_Sales
ON dbo.Sales
REBUILD PARTITION = <partition_number>;
```
✔ Rebuild only affected partitions

---

## 7. Validate Statistics Freshness
Required only if:
- Data movement occurred
- Large volume of rows was deleted or switched
```sql
SELECT
    s.name AS StatName,
    STATS_DATE(s.object_id, s.stats_id) AS LastUpdated
FROM sys.stats s
WHERE s.object_id = OBJECT_ID('dbo.Sales');
```
Optional refresh:
```sql
UPDATE STATISTICS dbo.Sales WITH FULLSCAN;
```

---

## 8. Verify Filegroup Space Usage
Ensure new filegroups have sufficient free space.
```sql
SELECT
    fg.name AS FilegroupName,
    df.name AS DataFile,
    df.size / 128.0 AS SizeMB,
    df.max_size,
    df.growth
FROM sys.filegroups fg
JOIN sys.database_files df
    ON fg.data_space_id = df.data_space_id
ORDER BY fg.name;
```
✔ Expected:
- Data files exist for new partitions
- Autogrowth settings are appropriate

---

## 9. Confirm No Data Leakage Between Partitions
Spot-check logical correctness using partition key predicates.
```sql
SELECT COUNT(*) AS UnexpectedRows
FROM dbo.Sales
WHERE SalesDate < '2025-01-01'
  AND $PARTITION.pf_SalesDate(SalesDate) >= <partition_number_for_2025_01>;
```
Result should be 0

---

## 10. Final Sanity Checks
- [ ] No active blocking during SPLIT / SWITCH
- [ ] No unexpected log growth
- [ ] Application queries unaffected
- [ ] Partition elimination working as expected



