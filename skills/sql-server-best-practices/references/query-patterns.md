# T-SQL & Query Patterns

## Stored Procedures

Always include at the top:
```sql
SET NOCOUNT ON;       -- Suppresses "N rows affected" noise, improves performance with ORMs
SET XACT_ABORT ON;    -- Rolls back transaction automatically on runtime error
```

Use `TRY/CATCH` for error handling in any proc that modifies data:
```sql
BEGIN TRY
    BEGIN TRANSACTION;
    -- DML here
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    THROW;  -- Re-raises the original error with original line/message
END CATCH;
```

Prefer `THROW` over `RAISERROR` — it's cleaner and preserves the original error number.

## SARGability (Search ARGument-able)

A predicate is SARGable when SQL Server can use an index seek rather than a scan.
This is one of the most common performance killers in T-SQL.

**Non-SARGable — forces a full scan:**
```sql
WHERE YEAR(OrderDate) = 2024
WHERE LEFT(CustomerCode, 3) = 'ABC'
WHERE CONVERT(VARCHAR, OrderId) = '12345'
WHERE ISNULL(Status, 'Active') = 'Active'
```

**SARGable equivalents:**
```sql
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
WHERE CustomerCode LIKE 'ABC%'
WHERE OrderId = 12345
WHERE Status = 'Active' OR Status IS NULL
```

The rule: **never wrap the indexed column in a function or implicit conversion.**

## Implicit Conversions

When comparing columns of different data types, SQL Server silently converts one side.
This can invalidate index seeks and cause poor cardinality estimates.

```sql
-- Bad: CustomerCode is NVARCHAR, this literal is VARCHAR — causes implicit conversion
WHERE CustomerCode = 'ABC123'

-- Good: match the data type
WHERE CustomerCode = N'ABC123'
```

Always check `sys.dm_exec_query_stats` or execution plans for the `CONVERT_IMPLICIT` warning.

## CTEs vs Subqueries vs Temp Tables

| Approach | Use when |
|----------|----------|
| CTE | One-time reference to a result; improves readability; recursive queries |
| Subquery | Simple, inline, non-reused logic |
| Temp table (`#temp`) | Result is used more than once; large intermediate result that benefits from statistics; need an index on intermediate data |
| Table variable (`@table`) | Small result sets (<1,000 rows); no need for statistics; short-lived in a proc |

Temp tables (`#temp`) get their own statistics and can be indexed — prefer them over table variables for anything non-trivial. On SQL Server 2019+ and Azure SQL, temp table deferred compilation reduces parameter sniffing issues.

```sql
-- Temp table with an index for a multi-use intermediate result
CREATE TABLE #Orders (
    OrderId   INT NOT NULL PRIMARY KEY,
    CustomerId INT NOT NULL,
    Total     DECIMAL(18,2) NOT NULL
);
CREATE INDEX IX_Orders_CustomerId ON #Orders (CustomerId);
```

## Parameter Sniffing

SQL Server compiles a plan for a stored proc using the parameter values from the *first* execution.
If those values are atypical (e.g., a customer with 500,000 orders when most have 5), every subsequent execution uses a bad plan.

Detection: plan looks right for some inputs but terrible for others.

Mitigation options (in order of preference):
1. `OPTION (OPTIMIZE FOR UNKNOWN)` — use average statistics, not the sniffed value
2. `OPTION (RECOMPILE)` on the specific statement — recompile on every execution (CPU cost, good for reports)
3. Local variable trick — assign params to local variables before use (loses sniffing entirely)
4. Separate procs for wildly different data distributions

```sql
-- Option 1: good default for most cases
SELECT * FROM dbo.Orders WHERE CustomerId = @CustomerId
OPTION (OPTIMIZE FOR UNKNOWN);

-- Option 2: use when the query runs infrequently and data varies a lot
SELECT * FROM dbo.Orders WHERE CustomerId = @CustomerId
OPTION (RECOMPILE);
```

## Cursors and Row-by-Row Processing

Cursors are almost always the wrong choice. Before writing one, ask: can this be done with a JOIN, window function, or UPDATE...FROM?

**When a cursor is actually justified:**
- Calling a stored proc once per row (no set-based equivalent)
- Complex multi-step logic that genuinely can't be vectorized
- Administrative scripts (not production query paths)

If you must use a cursor, use the most restrictive options:
```sql
DECLARE cur CURSOR
    LOCAL        -- scoped to the batch
    STATIC       -- snapshot, no locks held
    READ_ONLY
    FORWARD_ONLY
FOR SELECT ...;
```

## Window Functions Over Self-Joins

Window functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, `SUM() OVER`) are usually faster and clearer
than self-joins for row-numbering, running totals, or "previous/next row" logic.

```sql
-- Get the most recent order per customer
SELECT CustomerId, OrderId, OrderDate
FROM (
    SELECT CustomerId, OrderId, OrderDate,
           ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS rn
    FROM dbo.Orders
) x
WHERE rn = 1;
```

## NULL Handling

`NULL` is not a value — it's the absence of one. Three-valued logic (`TRUE/FALSE/UNKNOWN`) surprises many developers.

- `NULL = NULL` is `UNKNOWN`, not `TRUE` — use `IS NULL` / `IS NOT NULL`
- `NOT IN` with a subquery that returns any NULL will return zero rows — use `NOT EXISTS` instead
- `ISNULL(col, default)` is SQL Server-specific; `COALESCE(col, default)` is ANSI standard and handles multiple args

```sql
-- This returns zero rows if any CustomerIds in the subquery are NULL
SELECT * FROM dbo.Orders WHERE CustomerId NOT IN (SELECT CustomerId FROM dbo.Blacklist);

-- Safe equivalent
SELECT * FROM dbo.Orders o
WHERE NOT EXISTS (SELECT 1 FROM dbo.Blacklist b WHERE b.CustomerId = o.CustomerId);
```

## Pagination

Use `OFFSET/FETCH` (SQL Server 2012+) rather than the `ROW_NUMBER()` subquery pattern:
```sql
SELECT OrderId, OrderDate, Total
FROM dbo.Orders
ORDER BY OrderDate DESC
OFFSET (@Page - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

Ensure the `ORDER BY` column(s) have a supporting index for large tables — otherwise each page requires a sort of the full result.

## Dynamic SQL

When dynamic SQL is unavoidable, always use `sp_executesql` with parameterized queries. Never concatenate user input directly into a SQL string.

```sql
-- BAD — SQL injection risk
SET @sql = 'SELECT * FROM dbo.Orders WHERE Status = ''' + @status + '''';
EXEC(@sql);

-- GOOD — parameterized
SET @sql = N'SELECT * FROM dbo.Orders WHERE Status = @status';
EXEC sp_executesql @sql, N'@status NVARCHAR(50)', @status = @status;
```

See `references/security.md` for more on dynamic SQL and injection prevention.
