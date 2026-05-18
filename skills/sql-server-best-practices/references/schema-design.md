# Schema & Data Modeling

## Naming Conventions

Consistent naming reduces cognitive load and prevents subtle bugs (e.g., joining on the wrong column).

| Object | Convention | Example |
|--------|-----------|---------|
| Table | PascalCase, singular noun | `dbo.Order`, `dbo.CustomerAddress` |
| Column | PascalCase | `OrderDate`, `CustomerId` |
| Primary key | `Id` (or `TableNameId` for clarity in joins) | `OrderId` |
| Foreign key | Referenced table name + `Id` | `CustomerId` in `Order` table |
| Index | `IX_TableName_Columns` | `IX_Order_CustomerId` |
| Unique constraint | `UQ_TableName_Columns` | `UQ_Customer_Email` |
| Primary key constraint | `PK_TableName` | `PK_Order` |
| Foreign key constraint | `FK_ChildTable_ParentTable` | `FK_Order_Customer` |
| Stored procedure | `usp_` prefix + verb + noun | `usp_GetOrdersByCustomer` |
| View | `v_` prefix | `v_ActiveOrders` |
| Scalar function | `fn_` prefix | `fn_CalculateDiscount` |
| Schema | Lowercase | `dbo`, `reporting`, `staging` |

Avoid generic names: `Data`, `Info`, `Table1`, `Type` (conflicts with T-SQL keywords). Never prefix table names with `tbl_` — it's redundant.

## Primary Keys and Identity

Every table must have a primary key. Use integer identity columns as the default:

```sql
CREATE TABLE dbo.Product (
    ProductId   INT           NOT NULL IDENTITY(1,1),
    Name        NVARCHAR(200) NOT NULL,
    -- ...
    CONSTRAINT PK_Product PRIMARY KEY CLUSTERED (ProductId)
);
```

Per CLAUDE.md: all data models MUST use integer IDs as the primary identifier, auto-incremented. Each entity should have a `Name` or `Description` field.

**When to use UNIQUEIDENTIFIER (GUID):**
- Distributed systems that merge data from multiple sources
- Exposed in APIs where sequential IDs would be a security concern (use `NEWSEQUENTIALID()` as default, not `NEWID()`)

**Avoid**: `BIGINT` unless you genuinely expect > 2 billion rows; composite natural-key PKs (they make FKs verbose and joins slow).

## Data Types

Choose the smallest type that fits the data — it reduces storage, improves cache efficiency, and speeds up comparisons.

| Use case | Recommended type | Avoid |
|----------|-----------------|-------|
| Character data | `NVARCHAR(n)` for Unicode; `VARCHAR(n)` for ASCII-only | `NVARCHAR(MAX)` unless truly unbounded |
| Whole numbers | `INT` (default), `BIGINT` if > 2B rows, `TINYINT`/`SMALLINT` for lookup tables | `FLOAT` or `REAL` for counts |
| Money / prices | `DECIMAL(18, 4)` or `DECIMAL(19, 4)` | `MONEY` (rounding issues with arithmetic), `FLOAT` (binary imprecision) |
| Dates | `DATE` (date only), `DATETIME2(0-7)` (date + time) | `DATETIME` (legacy, less precise), `SMALLDATETIME` |
| Boolean flags | `BIT NOT NULL DEFAULT 0` | `CHAR(1)`, `INT`, `TINYINT` |
| Fixed-length codes | `CHAR(n)` or `NCHAR(n)` | `VARCHAR` with a CHECK constraint enforcing length (still fine, just less clear) |

**`NVARCHAR(MAX)` caution:** MAX columns can't be indexed directly, are stored off-row above 8,000 bytes, and prevent row compression. Only use when strings genuinely exceed 4,000 characters.

## Constraints

Define constraints in the database, not just in application code. The database is the last line of defense for data integrity.

```sql
CREATE TABLE dbo.Order (
    OrderId     INT             NOT NULL IDENTITY(1,1),
    CustomerId  INT             NOT NULL,
    Status      NVARCHAR(20)    NOT NULL DEFAULT N'Pending',
    Total       DECIMAL(18,4)   NOT NULL DEFAULT 0,
    OrderDate   DATETIME2(0)    NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Order                 PRIMARY KEY CLUSTERED (OrderId),
    CONSTRAINT FK_Order_Customer        FOREIGN KEY (CustomerId) REFERENCES dbo.Customer(CustomerId),
    CONSTRAINT CK_Order_Status          CHECK (Status IN (N'Pending', N'Processing', N'Shipped', N'Cancelled')),
    CONSTRAINT CK_Order_TotalNonNeg     CHECK (Total >= 0)
);
```

Always name your constraints — unnamed constraints get system-generated names like `CK__Order__Total__3A81B327` that are impossible to reference in error messages or migrations.

## Foreign Keys

Always create FK constraints on relationships. They:
- Prevent orphaned records at the database level
- Allow the optimizer to simplify join plans (FK elimination)
- Serve as self-documenting relationships

Index foreign key columns on the child table — SQL Server does not do this automatically, and unindexed FKs cause full scans during cascading deletes and joins.

```sql
-- After creating the FK, always add an index on the FK column(s)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId ON dbo.Order (CustomerId);
```

## NULL vs NOT NULL

Default to `NOT NULL` and use `DEFAULT` values to avoid null surprises. Only use `NULL` when "no value" is a genuine business state distinct from any default.

Excessive NULLs lead to:
- Three-valued logic bugs (see `query-patterns.md`)
- Inaccurate `COUNT(col)` results
- Wasted storage (nullable columns still consume bytes even with sparse data)

## Schema Separation

Use schemas to group related objects and to apply permissions cleanly:
- `dbo` — core application tables
- `reporting` — views and procs for reporting/BI consumers
- `staging` — ETL landing tables (should be truncated/reloaded, not permanent)
- `audit` — audit log tables and triggers
- `security` — permission-related objects

This also makes it easy to grant a reporting user `EXECUTE` on the `reporting` schema without touching core tables.

## Avoid Anti-Patterns

**EAV (Entity-Attribute-Value) tables:** `(EntityId, AttributeName, Value)` schemas are flexible but destroy query performance, type safety, and referential integrity. Use JSON columns (`NVARCHAR(MAX)` with `ISJSON` check) or proper schema changes instead.

**"God" tables:** A 200-column table with 90% NULLs on any given row is a sign that several entities have been collapsed into one. Normalize them.

**Storing delimited lists in a column:** `Tags = 'red,blue,green'` prevents indexing and joining. Use a child table with a proper FK, or use JSON if the structure is truly dynamic.

**`SELECT *` in views and procs:** Column additions/reorders can silently break dependent code. Always list columns explicitly.

## Versioning and Migrations

- Keep schema changes in source-controlled migration scripts, numbered sequentially
- Never make ad-hoc schema changes directly in production
- Test migrations against a production-size copy of data — DDL on large tables can lock or take hours
- Use `WITH (ONLINE = ON)` for index operations on Enterprise/Azure SQL to avoid blocking
- For large column additions with a default, on SQL Server 2012+ this is metadata-only (instant) when the default is a constant and the column is nullable or has a non-NULL default
