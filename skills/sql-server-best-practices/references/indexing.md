# Indexing Strategy

## Clustered Index

Every table should have a clustered index. It defines the physical storage order of the rows.

**Good clustered index candidates:**
- Integer identity column (most common, ideal for OLTP)
- Date column on time-series/append-only tables
- Natural key that is narrow, unique, ever-increasing, and used in range queries

**Avoid clustering on:**
- GUIDs (random insertion causes page fragmentation — use `NEWSEQUENTIALID()` if you must use GUIDs)
- Wide columns (all nonclustered indexes include the clustered key as a row locator — wide keys multiply storage)
- Frequently updated columns (updates to the clustered key are expensive)

```sql
-- Ideal: narrow, ever-increasing, integer
CREATE TABLE dbo.Orders (
    OrderId    INT IDENTITY(1,1) NOT NULL,
    ...
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderId)
);
```

## Nonclustered Indexes

Design nonclustered indexes around your queries, not your tables. The goal is to satisfy a query entirely from the index (a "covering index") without a key lookup into the clustered index.

**Columns in the key** (`ON table(col1, col2)`): used for seeks and range scans. Put the most selective equality predicate first, then range predicates.

**Included columns** (`INCLUDE(col3, col4)`): stored in the leaf level of the index only. Use for columns that appear in `SELECT` or `JOIN` but not in `WHERE`/`ORDER BY`. This avoids bloating the index key while still avoiding key lookups.

```sql
-- Query: SELECT OrderId, Total FROM dbo.Orders WHERE CustomerId = @id AND Status = 'Open'
-- Index: seek on CustomerId + Status, return Total without a key lookup
CREATE NONCLUSTERED INDEX IX_Orders_CustomerStatus
    ON dbo.Orders (CustomerId, Status)
    INCLUDE (Total);
```

### Index Key Column Order

The leading column rule: SQL Server can only seek into an index on a column if all preceding columns in the key are covered by equality predicates.

```
Index: (A, B, C)
WHERE A = 1             → seek on A ✓
WHERE A = 1 AND B = 2   → seek on A, B ✓
WHERE B = 2             → scan (A is not filtered) ✗
WHERE A = 1 AND C = 3   → seek on A only, filter on C ✓
```

Put high-selectivity equality columns first, then range columns.

## Filtered Indexes

Index only the rows matching a WHERE clause. Great for sparse data (e.g., an "active" flag where 95% of rows are inactive):

```sql
CREATE NONCLUSTERED INDEX IX_Orders_OpenStatus
    ON dbo.Orders (CustomerId)
    INCLUDE (Total, OrderDate)
    WHERE Status = 'Open';
```

The query must include the filter predicate for SQL Server to choose the filtered index.

## Missing and Unused Indexes

**Find missing indexes (from the DMVs):**
```sql
SELECT
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS ImprovementMeasure,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig  ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle  = mid.index_handle
ORDER BY ImprovementMeasure DESC;
```

Don't blindly add every suggested index — SQL Server doesn't account for write overhead or duplicate suggestions. Treat these as hints, not instructions.

**Find unused indexes (since last restart):**
```sql
SELECT
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.index_id,
    s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats s
    ON i.object_id = s.object_id AND i.index_id = s.index_id AND s.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.index_id > 0
  AND (s.user_seeks + s.user_scans + s.user_lookups = 0 OR s.user_seeks IS NULL)
ORDER BY s.user_updates DESC;
```

Every unused index still pays a write tax on INSERT/UPDATE/DELETE. Remove it if it hasn't been used since the last restart and the server has been running for a meaningful period (days/weeks, not hours).

## Index Maintenance: Rebuild vs Reorganize

Fragmentation degrades read performance because SQL Server must read more pages.

Check fragmentation:
```sql
SELECT
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 1000   -- ignore tiny indexes
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

| Fragmentation | Action |
|---------------|--------|
| < 10% | Do nothing |
| 10–30% | `ALTER INDEX ... REORGANIZE` (online, minimal locking) |
| > 30% | `ALTER INDEX ... REBUILD` (offline by default; use `WITH (ONLINE = ON)` on Enterprise/Azure SQL) |

Use Ola Hallengren's `IndexOptimize` for production maintenance — it makes these decisions automatically and logs results.

## Statistics

The query optimizer relies on statistics to estimate row counts. Stale statistics lead to bad plans.

- Auto-update statistics is ON by default — leave it on
- For very large tables, consider `AUTO_UPDATE_STATISTICS_ASYNC ON` at the database level to avoid blocking queries during a statistics update
- After large data loads, manually update stats: `UPDATE STATISTICS dbo.Orders WITH FULLSCAN;`
- On SQL Server 2016+ with compatibility level 130+, enable the new cardinality estimator (CE 2014+) — it handles skewed distributions better

## Key Lookups and Index Covering

A key lookup (RID lookup on heaps) in an execution plan is a warning sign — the optimizer is using a nonclustered index for the seek but then going back to the clustered index for additional columns. This is expensive at scale.

Fix: add the missing columns to the nonclustered index's `INCLUDE` clause.

```
Execution plan shows: Index Seek → Key Lookup
Solution: ALTER INDEX ... REBUILD with INCLUDE columns added,
          or DROP and recreate with broader INCLUDE list.
```

## Slow Query Diagnosis Workflow

Follow this sequence when a query is slow. Don't skip steps — each one narrows the cause.

**Step 1 — Get the actual execution plan**
Run with `SET STATISTICS IO ON` and capture the actual (not estimated) plan in SSMS:
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- your query here
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```
In the Messages tab, look for tables with high **logical reads** — that's where the problem is.

**Step 2 — Read the execution plan for warning signs**
In order of severity:
- **Key Lookup / RID Lookup** → missing `INCLUDE` columns on a nonclustered index
- **Table Scan / Clustered Index Scan** on a large table → no usable index, or non-SARGable predicate
- **Sort** with high cost → missing index on `ORDER BY` column, or large result being sorted
- **Hash Match** on joins with large row counts → missing index on join column(s), or bad cardinality estimate
- **Warning triangle** on any operator → hover to read (implicit conversions, missing stats, spill to TempDB)

**Step 3 — Check for SARGability issues**
See `query-patterns.md` — functions on indexed columns and implicit type conversions are the most common cause of unexpected scans.

**Step 4 — Check missing index suggestions**
The execution plan may show a green "Missing Index" suggestion. Also query the DMVs (see "Missing and Unused Indexes" section above). Don't add without reviewing — the suggestions ignore write overhead.

**Step 5 — Check for parameter sniffing**
Parameter sniffing is one of the most common causes of "runs fine sometimes, terrible other times" or "fast in SSMS, slow from the app." Both symptoms point to the compiled plan being optimized for the wrong parameter values.

Signs: the query has been running fine for months, then suddenly degrades after a restart or recompile. Or it runs in 10ms from SSMS with `EXEC usp_GetOrders 12345` but takes 30 seconds from the application.

See the Parameter Sniffing section in `query-patterns.md` for mitigation options including `OPTIMIZE FOR UNKNOWN`, `OPTION (RECOMPILE)`, and the local variable trick (a legitimate last resort when other options fail).

**Step 6 — Use sp_BlitzCache to find the worst offenders system-wide**
See `maintenance.md` for `sp_BlitzCache` usage. Sort by `reads` or `duration` to find the queries burning the most resources.

## Heaps (Tables Without a Clustered Index)

Avoid heaps for tables that are frequently read or updated. They cause:
- Forwarding pointers on updates (row moves when the row grows)
- Full scans on most reads
- Excessive fragmentation

The only justifiable heap is a staging/bulk-load table where you insert once and truncate frequently.
