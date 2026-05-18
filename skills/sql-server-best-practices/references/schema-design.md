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

Every table must have a primary key. Use a single-column **surrogate key** — either `INT IDENTITY` or `UNIQUEIDENTIFIER` with `NEWSEQUENTIALID()`. Never use composite natural keys (email + phone, name + date, etc.) as a PK.

### INT IDENTITY — recommended for most OLTP

```sql
CREATE TABLE dbo.Product (
    ProductId   INT           NOT NULL IDENTITY(1,1),
    Name        NVARCHAR(200) NOT NULL,
    CONSTRAINT PK_Product PRIMARY KEY CLUSTERED (ProductId)
);
```

`IDENTITY(1,1)` means: start at 1, increment by 1 — the database assigns the next value automatically on every insert. You never set it manually. This is what makes it "auto-increment."

**Why INT IDENTITY is the SQL Server DBA community default:**
- 4 bytes — compact in the clustered index and in every nonclustered index that carries it as a row locator
- Sequential inserts always land at the end of the B-tree — zero page fragmentation
- Readable and debuggable in logs, support tickets, and conversations
- Use `BIGINT` only if you genuinely expect > 2.1 billion rows; `INT` covers virtually all OLTP tables

### GUID — right for distributed and cloud scenarios

```sql
CREATE TABLE dbo.Product (
    ProductId   UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    Name        NVARCHAR(200)    NOT NULL,
    CONSTRAINT PK_Product PRIMARY KEY CLUSTERED (ProductId)
);
```

Use `NEWSEQUENTIALID()` as the default — it generates GUIDs in ascending order so inserts still land at the end of the clustered index. **Never use `NEWID()`** as a clustered index key — random GUIDs scatter inserts across the B-tree causing severe fragmentation.

**When GUIDs are the right choice:**
- Data merges across multiple databases or servers (GUIDs are globally unique by definition)
- Microservices where IDs are generated client-side before the row is inserted
- APIs where sequential integer IDs would allow enumeration by attackers
- Azure SQL with geo-replication or sync scenarios

GUIDs are 16 bytes vs 4 — every nonclustered index is larger and they are harder to read, but these are acceptable tradeoffs in the scenarios above.

### What to always avoid

- **Composite natural keys** (`CustomerCode + OrderDate`, `Email + Phone`): natural values change, they propagate into every FK, and joins become verbose
- **Random GUIDs (`NEWID()`) on clustered indexes**: causes fragmentation — use `NEWSEQUENTIALID()` instead
- **String PKs**: change over time, type-unsafe, slow to compare and join

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

Index foreign key columns on the child table — SQL Server does not do this automatically, and unindexed FKs cause full scans during joins and deletes.

```sql
-- After creating the FK, always add an index on the FK column(s)
CREATE NONCLUSTERED INDEX IX_Order_CustomerId ON dbo.Order (CustomerId);
```

### Do Not Use Cascade Deletes

**Never use `ON DELETE CASCADE`.** It is one of the most dangerous options in SQL Server schema design:

- A single `DELETE` on a parent row silently removes an unbounded number of child rows across one or more tables — with no warning, no confirmation, and no easy undo
- Cascades chain: if A cascades to B, and B cascades to C, deleting one row in A can wipe out entire subtrees of data invisibly
- Audit logs show only the parent delete — the child row destruction leaves no trace in standard logging
- They create tight coupling between tables that makes schema changes risky
- SQL Server will reject circular cascade paths entirely, which can force awkward schema workarounds

**Use explicit deletes instead.** Handle related-row cleanup in a stored procedure or application transaction where the intent is visible and the scope is controlled:

```sql
-- BAD — invisible destruction
ALTER TABLE dbo.OrderItem
    ADD CONSTRAINT FK_OrderItem_Order
    FOREIGN KEY (OrderId) REFERENCES dbo.Order(OrderId)
    ON DELETE CASCADE;  -- Never do this

-- GOOD — explicit, auditable, intentional
CREATE OR ALTER PROCEDURE dbo.usp_DeleteOrder
    @OrderId INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    BEGIN TRANSACTION;
        DELETE FROM dbo.OrderItem  WHERE OrderId = @OrderId;
        DELETE FROM dbo.OrderNote  WHERE OrderId = @OrderId;
        DELETE FROM dbo.Order      WHERE OrderId = @OrderId;
    COMMIT TRANSACTION;
END;
```

The default FK behavior (`NO ACTION`) is correct — let the database reject deletes that would orphan child rows, and force the caller to clean up explicitly.

`ON DELETE SET NULL` has similar problems (silently corrupts FK relationships) and should also be avoided. `ON UPDATE CASCADE` is occasionally reasonable for natural-key schemas but is rarely needed with integer identity PKs.

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

**`ON DELETE CASCADE`:** See the Foreign Keys section above. Never use it. The silent mass-deletion risk far outweighs the convenience.

**Composite natural keys or string PKs:** Use a surrogate key (`INT IDENTITY` or `UNIQUEIDENTIFIER` with `NEWSEQUENTIALID()`) instead. Natural values change, string comparisons are slower, and composite keys propagate every column into every FK. See the Primary Keys section above for guidance on INT vs GUID.

## Versioning and Migrations

- Keep schema changes in source-controlled migration scripts, numbered sequentially
- Never make ad-hoc schema changes directly in production
- Test migrations against a production-size copy of data — DDL on large tables can lock or take hours
- Use `WITH (ONLINE = ON)` for index operations on Enterprise/Azure SQL to avoid blocking
- For large column additions with a default, on SQL Server 2012+ this is metadata-only (instant) when the default is a constant and the column is nullable or has a non-NULL default
