# Security

## Principle of Least Privilege

Grant only the permissions a login or user actually needs. Never use `sa`, `dbo`, or `db_owner` for application connections.

**Typical application account setup:**
```sql
-- Create a login at the server level
CREATE LOGIN AppLogin WITH PASSWORD = N'<strong-password>';

-- Create a user in the application database mapped to that login
USE AppDatabase;
CREATE USER AppUser FOR LOGIN AppLogin;

-- Grant only what the app needs
GRANT SELECT, INSERT, UPDATE ON SCHEMA::dbo TO AppUser;
-- If the app calls stored procs, grant EXECUTE on the schema (not on individual tables)
GRANT EXECUTE ON SCHEMA::dbo TO AppUser;
-- Revoke direct table access if all access is through procs
REVOKE SELECT, INSERT, UPDATE ON SCHEMA::dbo FROM AppUser;
```

**For read-only connections** (reporting, dashboards):
```sql
-- ALTER ROLE is the current syntax (sp_addrolemember is deprecated since SQL Server 2012)
ALTER ROLE db_datareader ADD MEMBER ReportingUser;
-- Or use a dedicated reporting schema for tighter scoping:
GRANT SELECT ON SCHEMA::reporting TO ReportingUser;
```

Separate logins for: application writes, application reads, reporting, ETL, and DBAs.

## Authentication: Windows vs SQL

**Prefer Windows Authentication for on-premises SQL Server.** It is more secure because:
- No password stored in SQL Server (Kerberos/NTLM handles it)
- Subject to Active Directory password policies, account lockout, and group membership
- Easier to audit — logs show the actual Windows identity

```sql
-- Windows login (no password — AD manages it)
CREATE LOGIN [DOMAIN\AppServiceAccount] FROM WINDOWS;
```

Use SQL Authentication only when:
- Connecting from a non-Windows client or cross-forest without trust
- Azure SQL (Windows auth requires Azure AD integration — use that, not basic SQL auth where possible)
- Local development/testing

**Azure SQL:** Use Azure Active Directory (Entra ID) authentication instead of SQL logins — it supports MFA, conditional access, and managed identities (no stored credentials at all):
```sql
-- Grant a managed identity access to Azure SQL
CREATE USER [my-app-managed-identity] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [my-app-managed-identity];
```

## SQL Injection Prevention

SQL injection is the #1 database vulnerability. The fix is consistent parameterization.

**Always use parameterized queries from application code.** Never build SQL strings by concatenating user input.

```csharp
// BAD — injectable
var sql = $"SELECT * FROM Users WHERE Username = '{username}'";

// GOOD — parameterized (ADO.NET)
var cmd = new SqlCommand("SELECT * FROM dbo.Users WHERE Username = @username", conn);
cmd.Parameters.AddWithValue("@username", username);
```

**For dynamic SQL inside T-SQL**, always use `sp_executesql` with parameters (never `EXEC(@string)`):
```sql
-- BAD
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM dbo.Orders WHERE Status = ''' + @status + N'''';
EXEC(@sql);

-- GOOD
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM dbo.Orders WHERE Status = @status';
EXEC sp_executesql @sql, N'@status NVARCHAR(20)', @status = @status;
```

**Validating dynamic object names** (table/column names can't be parameterized):
```sql
-- Use QUOTENAME() to safely bracket user-supplied identifiers
SET @sql = N'SELECT * FROM ' + QUOTENAME(@tableName);
-- But also whitelist against known-good values first
IF @tableName NOT IN ('Orders', 'Customers', 'Products')
    THROW 50001, 'Invalid table name.', 1;
```

## Stored Procedure Ownership Chaining

A key security benefit of stored procedures: the caller needs `EXECUTE` on the proc but does NOT need direct table permissions, as long as the proc and its tables are owned by the same schema/user.

This means you can expose a "surface" of only the operations the application should be able to perform, while the underlying tables are inaccessible directly.

```sql
-- User only has EXECUTE on dbo — cannot SELECT from dbo.Orders directly
-- but can call usp_GetCustomerOrders which reads dbo.Orders
GRANT EXECUTE ON SCHEMA::dbo TO AppUser;
REVOKE SELECT ON dbo.Orders FROM AppUser;
```

## Encryption at Rest

**Transparent Data Encryption (TDE):** Encrypts the entire database file at rest. Transparent to the application — no query changes needed.

```sql
-- Enable TDE (requires a certificate in master first)
USE master;
CREATE CERTIFICATE TDECert WITH SUBJECT = N'TDE Certificate';

USE AppDatabase;
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDECert;
ALTER DATABASE AppDatabase SET ENCRYPTION ON;
```

On Azure SQL, TDE is enabled by default with service-managed keys. Consider customer-managed keys (CMK) for compliance requirements.

**Always Encrypted:** Protects sensitive columns (SSNs, credit card numbers) even from DBAs — the encryption/decryption happens in the client driver. Appropriate for the highest-sensitivity columns; has query limitations (can only filter on deterministic encryption, not randomized).

**Column-level encryption with `ENCRYPTBYKEY`:** Less overhead than Always Encrypted, but the encryption key is still accessible to DBAs. Use when you want encryption-at-rest for specific columns without the client-driver requirement.

## Auditing

Enable auditing for:
- All logins to production (failed logins especially)
- DDL changes (schema changes, permission grants)
- SELECT/DML on sensitive tables (PII, financial data)

SQL Server Audit (built-in, 2008+):
```sql
-- Server-level audit target
CREATE SERVER AUDIT AppAudit TO FILE (FILEPATH = N'C:\Audits\');
ALTER SERVER AUDIT AppAudit WITH (STATE = ON);

-- Database-level audit specification
CREATE DATABASE AUDIT SPECIFICATION AuditSensitiveData
FOR SERVER AUDIT AppAudit
ADD (SELECT, INSERT, UPDATE, DELETE ON dbo.PatientRecord BY PUBLIC);
ALTER DATABASE AUDIT SPECIFICATION AuditSensitiveData WITH (STATE = ON);
```

On Azure SQL, use Auditing in the Azure portal — it writes to a Storage Account, Log Analytics, or Event Hub.

## Additional Hardening

**Disable `xp_cmdshell`** unless explicitly required:
```sql
EXEC sp_configure 'xp_cmdshell', 0;
RECONFIGURE;
```

**Disable remote procedure calls and linked servers** if not needed — they expand the attack surface.

**sa account:** Disable it or rename it. Use named admin accounts so audits show who actually connected.

**Password policy:** Enforce `CHECK_POLICY = ON`, `CHECK_EXPIRATION = ON` for all SQL logins.

**Contained databases:** If using contained database users, they bypass server-level logins — understand the implications before enabling.

**Row-Level Security (RLS):** For multi-tenant databases where each tenant should only see their own rows, RLS enforces this transparently without requiring every query to include a tenant filter:
```sql
CREATE FUNCTION dbo.fn_RLSFilter (@TenantId INT)
RETURNS TABLE WITH SCHEMABINDING
AS RETURN SELECT 1 AS Result WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);

CREATE SECURITY POLICY TenantFilter
ADD FILTER PREDICATE dbo.fn_RLSFilter(TenantId) ON dbo.Orders;
```
