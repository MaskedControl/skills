# Maintenance & Monitoring

## Health Check: sp_Blitz (First Responder Kit)

`sp_Blitz` is the fastest way to get a prioritized list of SQL Server configuration and health issues.
Install from: https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit

```sql
-- Basic health check — returns prioritized findings
EXEC sp_Blitz;

-- Only show urgent issues (Priority 50 and below)
EXEC sp_Blitz @IgnorePrioritiesAbove = 50;

-- Write to a table for tracking over time
EXEC sp_Blitz
    @OutputDatabaseName = 'DBAtools',
    @OutputSchemaName   = 'dbo',
    @OutputTableName    = 'BlitzResults';
```

Run `sp_Blitz` after every new server setup and periodically in production (weekly or after major changes).

## Query Performance: sp_BlitzCache

Identifies the most resource-intensive queries currently in the plan cache.

```sql
-- Top queries by CPU
EXEC sp_BlitzCache;

-- Top queries by different metrics
EXEC sp_BlitzCache @SortOrder = 'reads';        -- Logical reads
EXEC sp_BlitzCache @SortOrder = 'duration';     -- Elapsed time
EXEC sp_BlitzCache @SortOrder = 'spills';       -- TempDB spills (memory pressure indicator)
EXEC sp_BlitzCache @SortOrder = 'memory grant'; -- Large memory grants

-- Filter to a specific database or stored proc
EXEC sp_BlitzCache @DatabaseName = 'AppDatabase';
EXEC sp_BlitzCache @StoredProcName = 'usp_GetOrders';

-- More results, more columns
EXEC sp_BlitzCache @Top = 20, @ExpertMode = 1;
```

## Index Analysis: sp_BlitzIndex

```sql
-- Analyze all indexes in the current database
EXEC sp_BlitzIndex;

-- Deep dive on a specific table
EXEC sp_BlitzIndex
    @DatabaseName = 'AppDatabase',
    @SchemaName   = 'dbo',
    @TableName    = 'Orders';

-- Show only indexes with zero reads (candidates for removal)
EXEC sp_BlitzIndex @Filter = 1;
```

## Wait Statistics: sp_BlitzFirst

Shows what SQL Server is *currently* waiting on — the fastest way to diagnose active performance problems.

```sql
-- Snapshot of current waits
EXEC sp_BlitzFirst;

-- 30-second sample (more accurate for intermittent issues)
EXEC sp_BlitzFirst @Seconds = 30;
```

Common waits and their meaning:
| Wait type | Likely cause |
|-----------|-------------|
| `CXPACKET` / `CXCONSUMER` | Parallelism — may need `MAXDOP` tuning |
| `LCK_M_*` | Blocking/locking — check for long-running transactions |
| `PAGEIOLATCH_SH` | Reading from disk — storage I/O bottleneck or missing indexes |
| `SOS_SCHEDULER_YIELD` | CPU pressure — expensive queries or insufficient CPU |
| `WRITELOG` | Log flush latency — slow storage for the transaction log |
| `RESOURCE_SEMAPHORE` | Memory grants — queries requesting more memory than available |

## Ola Hallengren Maintenance Solution

The gold standard for automated SQL Server maintenance. Install from: https://ola.hallengren.com

**Index optimization** (run nightly or weekly depending on fragmentation rate):
```sql
EXEC dbo.IndexOptimize
    @Databases = 'AppDatabase',
    @FragmentationLow  = NULL,            -- skip if <10%
    @FragmentationMedium = 'INDEX_REORGANIZE',
    @FragmentationHigh   = 'INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
    @FragmentationLevel1 = 10,
    @FragmentationLevel2 = 30,
    @UpdateStatistics = 'ALL',
    @OnlyModifiedStatistics = 'Y',
    @LogToTable = 'Y';
```

**Integrity check** (run weekly, never skip CHECKDB):
```sql
EXEC dbo.DatabaseIntegrityCheck
    @Databases = 'ALL_DATABASES',
    @CheckCommands = 'CHECKDB',
    @LogToTable = 'Y';
```

**Backups** (schedule via SQL Agent):
```sql
-- Full backup (weekly)
EXEC dbo.DatabaseBackup
    @Databases      = 'ALL_DATABASES',
    @Directory      = N'\\backup-server\SQLBackups',
    @BackupType     = 'FULL',
    @Verify         = 'Y',
    @CleanupTime    = 168,    -- Keep 7 days (hours)
    @LogToTable     = 'Y';

-- Transaction log backup (every 15 minutes for RPO)
EXEC dbo.DatabaseBackup
    @Databases      = 'ALL_DATABASES',
    @Directory      = N'\\backup-server\SQLBackups',
    @BackupType     = 'LOG',
    @Verify         = 'Y',
    @CleanupTime    = 24,
    @LogToTable     = 'Y';
```

## Backup Strategy

| Recovery Model | What it protects | Log backup required? |
|---------------|-----------------|---------------------|
| FULL | Point-in-time recovery | Yes |
| BULK_LOGGED | Same, with faster bulk ops | Yes |
| SIMPLE | Recover to last full/diff only | No |

Use `FULL` recovery model for any production database where data loss is unacceptable.
Use `SIMPLE` only for dev/staging or databases you can fully reload from source.

**Always verify your backups:**
```sql
RESTORE VERIFYONLY FROM DISK = N'\\backup-server\SQLBackups\AppDatabase_FULL_*.bak';
```

**Test restores regularly** — a backup you've never restored is not a real backup.

## DBCC CHECKDB

Detects and optionally repairs database corruption. Run at minimum weekly.

```sql
-- Check integrity (read-only, no repair)
DBCC CHECKDB ('AppDatabase') WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Physical-only check (faster, less thorough — good for very large databases)
DBCC CHECKDB ('AppDatabase') WITH PHYSICAL_ONLY, NO_INFOMSGS;
```

If CHECKDB reports errors, do not run REPAIR_ALLOW_DATA_LOSS without first restoring from backup — the repair operation loses data. CHECKDB corruption means restore from backup is the correct first response.

## Statistics Maintenance

Statistics drive query plan quality. They go stale as data changes.

```sql
-- Update all statistics in a database with a full scan
EXEC sp_updatestats;                                         -- sampling (fast)
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;                   -- full scan (accurate, slower)

-- Check when statistics were last updated
SELECT
    OBJECT_NAME(s.object_id) AS TableName,
    s.name AS StatName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) = 'Orders'
ORDER BY sp.last_updated ASC;
```

## TempDB Configuration

TempDB contention is a common bottleneck on busy servers.

**Number of data files:** Match to the number of logical CPU cores, up to 8. Beyond 8, add more only if you still see contention on `2:1:1` (PFS/GAM/SGAM pages).

```sql
-- Check current tempdb file count
SELECT name, physical_name, size * 8 / 1024 AS SizeMB
FROM tempdb.sys.database_files;
```

**Pre-size TempDB** to avoid autogrowth during peak load. Set all data files to equal size.

On Azure SQL, TempDB is managed — you don't configure file counts directly, but you can influence it by upgrading the service tier.

## Blocking and Deadlock Monitoring

**Find current blocking:**
```sql
EXEC sp_WhoIsActive;  -- Part of First Responder Kit / Adam Machanic
-- or built-in:
SELECT
    blocking_session_id,
    session_id,
    wait_type,
    wait_time / 1000.0 AS WaitSeconds,
    SUBSTRING(st.text, (r.statement_start_offset/2)+1, 
              ((CASE r.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
                ELSE r.statement_end_offset END - r.statement_start_offset)/2)+1) AS CurrentStatement
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE blocking_session_id > 0;
```

**Enable deadlock trace flag (on-prem):**
```sql
DBCC TRACEON(1222, -1);  -- Detailed deadlock XML in error log
```

On Azure SQL, deadlock graphs are available in `sys.event_log` / Extended Events automatically.
