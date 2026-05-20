# Agent Skills

A collection of Claude Code agent skills for SQL Server, database development, document generation, and more.

Skills follow the [Agent Skills](https://agentskills.io/) format and are installable via:

```bash
npx skills add MaskedControl/skills
```

---

## Available Skills

### sql-server-best-practices

SQL Server and Azure SQL best practices for developers and DBAs. Covers T-SQL patterns, indexing strategy, schema design, security, and maintenance — for both on-premises SQL Server 2016+ and Azure SQL.

**Use when:**

- Writing or reviewing T-SQL queries, stored procedures, or functions
- Designing or migrating database schemas
- Diagnosing slow queries, adding indexes, or reading execution plans
- Setting up database permissions or preventing SQL injection
- Configuring maintenance jobs, backups, or health checks
- Reviewing any SQL or DDL migration files

**Topics covered:**

- Query patterns (SARGability, set-based ops, CTEs, parameter sniffing, error handling, dynamic SQL)
- Indexing strategy (clustered vs. nonclustered, covering indexes, filtered indexes, fragmentation)
- Schema design (data types, naming conventions, constraints, foreign keys, anti-patterns)
- Security (least privilege, injection prevention, TDE, Always Encrypted, RLS, auditing)
- Maintenance & monitoring (sp_Blitz, sp_BlitzCache, Ola Hallengren, CHECKDB, backups, wait stats)

---

### create-docx

Generate Word documents (.docx) using docx-js on Windows. Covers the local npm setup pattern, running and cleaning up the build script, and reusable formatting components.

**Use when:**

- Creating a Word document for a client, email, or report
- Generating a .docx file programmatically from structured content
- Building documents with screenshots placeholders, callout boxes, or styled tables
- Reproducing a document from a previous session using the build script pattern

**Topics covered:**

- Windows-specific setup (local `npm install docx` pattern, why global install fails)
- Page size and margin setup (US Letter, DXA units, content width formula)
- Numbered lists and bullets (numbering config, table cell variant)
- Tables (dual-width requirement, DXA only, ShadingType.CLEAR)
- Screenshot placeholder component
- Callout box component
- Cleanup after generation
- Windows validator caveat

---

## Installation

```bash
npx skills add MaskedControl/skills
```

Select the skills you want to install when prompted.

## Skill Structure

Each skill contains:

- `SKILL.md` — Instructions for the agent, with topic routing
- `references/` — Supporting documentation loaded on demand (keeps the base skill lean)

## Contributing

Additional skills are welcome. Follow the structure above — one subdirectory per skill under `skills/`, with a `SKILL.md` and optional `references/` folder.

## License

MIT
