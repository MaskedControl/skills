---
name: sql-server-best-practices
description: >
  SQL Server and Azure SQL best practices for developers and DBAs. Use this skill whenever the user
  is writing T-SQL, designing schemas, asking about indexes, query performance, database security,
  or SQL Server maintenance — even if they don't explicitly mention "best practices." Triggers on
  requests like "write a stored proc," "should I use a cursor here," "how should I name this table,"
  "why is this query slow," "set up db permissions," or "how do I maintain indexes." Also use for
  code review of any SQL or schema migration files.
---

# SQL Server Best Practices

This skill covers both on-premises SQL Server (2016+) and Azure SQL Database / Managed Instance.
When differences exist between the two, they are called out in the relevant section.

## How to Use This Skill

Identify which domain(s) the user's request falls into, then read the corresponding reference file(s)
before responding. For requests that touch multiple domains (e.g., "review this stored proc and its
indexes"), read all relevant files.

| Domain | When to use | Reference file |
|--------|-------------|----------------|
| T-SQL & query patterns | Writing queries, stored procs, functions, CTEs, cursors, error handling | `references/query-patterns.md` |
| Indexing strategy | Index design, fragmentation, missing/duplicate indexes, query plan issues | `references/indexing.md` |
| Schema & data modeling | Table design, data types, naming conventions, constraints, relationships | `references/schema-design.md` |
| Security | Permissions, least privilege, dynamic SQL injection, encryption, auditing | `references/security.md` |
| Maintenance & monitoring | Health checks, index rebuild, statistics, backups, CHECKDB | `references/maintenance.md` |

## Quick Principles (always apply)

These apply regardless of domain — internalize them before reading any reference file:

1. **Correctness first.** A fast query that returns wrong results is worse than a slow correct one.
2. **Set-based over row-by-row.** SQL Server is optimized for set operations; cursors and loops are last resorts.
3. **Explicit over implicit.** Always specify schema (`dbo.TableName`), always use `SET NOCOUNT ON` in procs, always define column lists in `INSERT`.
4. **Test with realistic data volumes.** A query that looks fine on 1,000 rows can fall apart at 10 million.
5. **Don't guess — measure.** Use execution plans, `SET STATISTICS IO ON`, and `sp_BlitzCache` rather than intuiting performance.
6. **Least surprise.** Name things clearly. A reader unfamiliar with the system should be able to understand what a table, column, or proc does from its name alone.

## Determining Which Reference(s) to Read

Ask yourself what the user is trying to accomplish:

- **"Write/fix/review a query or stored proc"** → `query-patterns.md`
- **"Why is this slow / add an index / review query plan"** → `indexing.md` (and possibly `query-patterns.md`)
- **"Design a table / choose a data type / naming question"** → `schema-design.md`
- **"Set up permissions / prevent SQL injection / audit access"** → `security.md`
- **"Set up maintenance jobs / check database health / configure backups"** → `maintenance.md`
- **"Review this migration or DDL script"** → `schema-design.md` + `security.md`
- **"General code review of SQL"** → read all five, prioritize issues by severity
