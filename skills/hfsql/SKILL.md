---
name: hfsql
description: Interact with HFSQL databases (PC SOFT / WinDev) via the hfsql MCP server. Query, explore, insert, update, delete records in any HFSQL database.
---

# HFSQL Database Skill

You have access to an HFSQL MCP server that provides 8 tools for full database interaction via PowerShell ODBC.

## Tools Reference

All tools are prefixed with `mcp__hfsql__`. Always use the full name when calling them.

| Full tool name | Purpose | Required params |
|---|---|---|
| `mcp__hfsql__hfsql_connect` | Connect to a database via ODBC DSN | `id`, `dsn` (+ optional `uid`, `pwd`) |
| `mcp__hfsql__hfsql_disconnect` | Close a connection | `id` |
| `mcp__hfsql__hfsql_list_tables` | List tables with column counts | `id` (+ optional `with_row_count: true`) |
| `mcp__hfsql__hfsql_describe_table` | Get column schema for a table | `id`, `table` |
| `mcp__hfsql__hfsql_query` | Execute SELECT → JSON results | `id`, `sql` (+ optional `limit`) |
| `mcp__hfsql__hfsql_execute` | Execute INSERT / UPDATE / DELETE | `id`, `sql` |
| `mcp__hfsql__hfsql_insert` | Smart insert from JSON (auto-skips calculated fields, auto-formats dates) | `id`, `table`, `data` |
| `mcp__hfsql__hfsql_schema_summary` | Full schema dump with all columns, types, row counts | `id` |

## Step-by-step Workflow

### 1. Always connect first

```
mcp__hfsql__hfsql_connect(id: "mydb", dsn: "mydsn", uid: "admin", pwd: "admin")
```

- `id`: a friendly label you pick — reuse it in every subsequent call
- `dsn`: the ODBC Data Source Name configured in Windows ODBC Manager
- `uid` / `pwd`: default to `"admin"` / `"admin"` if omitted

### 2. Explore the schema

```
mcp__hfsql__hfsql_list_tables(id: "mydb", with_row_count: true)
mcp__hfsql__hfsql_describe_table(id: "mydb", table: "Articles")
```

### 3. Query data

```
mcp__hfsql__hfsql_query(id: "mydb", sql: "SELECT * FROM [Articles]", limit: 50)
```

### 4. Modify data

For raw SQL:
```
mcp__hfsql__hfsql_execute(id: "mydb", sql: "UPDATE [Articles] SET [Libellé] = 'NEW NAME' WHERE IDArticle = 1")
```

For smart insert (recommended for INSERTs):
```
mcp__hfsql__hfsql_insert(id: "mydb", table: "Articles", data: {"Reference": "M004", "Libellé": "BLÉ TENDRE", "Prix_Achat_HT": 4500})
```

### 5. Disconnect when done

```
mcp__hfsql__hfsql_disconnect(id: "mydb")
```

## HFSQL SQL Rules — CRITICAL

You MUST follow these rules or queries will fail:

### No SELECT without FROM
HFSQL has no DUAL table. `SELECT 1` will fail. Always use a real table:
```sql
SELECT COUNT(*) FROM [Articles]
```

### Square brackets for all identifiers
Always wrap table and column names in `[ ]`, especially those with French accents:
```sql
SELECT [Libellé], [Référence] FROM [Règlement] WHERE [Année] = 2025
```

### Date format: YYYYMMDD only
HFSQL rejects dashes and slashes in dates. Use strict `YYYYMMDD`:
```sql
WHERE [Date] = '20250517'
```
The `hfsql_insert` tool auto-converts `YYYY-MM-DD` → `YYYYMMDD`, but for `hfsql_execute` and `hfsql_query` you must format dates yourself.

### Calculated fields reject writes
Some columns are computed by HFSQL (e.g. `Mois` derived from `Date`). Inserting or updating them causes error `"rubrique calculée"`. The `hfsql_insert` tool auto-detects and skips these. For `hfsql_execute`, exclude them manually.

### ID columns are auto-increment
Primary keys follow the pattern `ID{TableName}` (e.g. `IDMouvement`, `IDArticle`). Never set them in INSERT — they are generated automatically.

### No INFORMATION_SCHEMA
HFSQL does not support `INFORMATION_SCHEMA`, `SHOW TABLES`, or `sys.tables`. Use `hfsql_list_tables` and `hfsql_describe_table` instead.

## HFSQL ERP Document Patterns

HFSQL is commonly used by WinDev ERP applications. The `Mouvement` table typically stores ALL document types (invoices, delivery notes, purchase orders, etc.) differentiated by a `Type` column. To find the type codes:

```
mcp__hfsql__hfsql_query(id: "mydb", sql: "SELECT [TYPE], [Définition], [Préfixe] FROM [Type_Pièce]")
```

### Creating a document

1. **Find the type code** from `Type_Pièce`
2. **Find the next number**: `SELECT MAX([Numéro]) FROM [Mouvement] WHERE [Type] = X AND [Année] = Y`
3. **Build the reference**: typically `{PREFIX}{YY}-{NUMBER:05d}` (e.g. `FV25-00020`)
4. **Insert the header** in `Mouvement` via `hfsql_insert`
5. **Insert line items** in `Ligne_MT` referencing the new `IDMouvement`
6. **Verify** with `hfsql_query`

### Linking documents

Documents are linked via `MT_Associer` (source → generated). Payments link via `Règlement` + `Ligne_RG`.

## Safety Rules

- **Always confirm** with the user before any INSERT / UPDATE / DELETE
- **Check fiscal year lock** (`Verrou` in `Exercice`) before modifying data for a given year — if `Verrou = 1`, the year is closed
- **Verify references** don't already exist before inserting new documents
- **Use `hfsql_query` to verify** after every modification
- **Never modify system tables** (`Paramètres`, `Société`) without explicit user request
- **Always use `hfsql_insert`** over raw `hfsql_execute` for INSERTs to avoid calculated field errors

## Error Handling

| Error message | Cause | Fix |
|---|---|---|
| `rubrique calculée` | Trying to write a calculated field | Remove the field from INSERT/UPDATE, or use `hfsql_insert` |
| `Fichier XXX inconnu` | Wrong table name | Check with `hfsql_list_tables` |
| Garbled date (e.g. `07/20/0005`) | Wrong date format | Use `YYYYMMDD` strictly |
| `Connexion "X" introuvable` | Forgot to connect first | Call `hfsql_connect` |
| `Error connecting to the database` | Wrong DSN, server down, or bad credentials | Verify DSN exists in ODBC Manager, check server is reachable on port 4900 |

## Remote Connections

To connect to a remote HFSQL server, create an ODBC DSN via registry:

```powershell
Add-OdbcDsn -Name "remote-db" -DriverName "HFSQL" -DsnType "User" -SetPropertyValue @()
Set-ItemProperty -Path "HKCU:\SOFTWARE\ODBC\ODBC.INI\remote-db" -Name "Server Name" -Value "192.168.1.102"
Set-ItemProperty -Path "HKCU:\SOFTWARE\ODBC\ODBC.INI\remote-db" -Name "Server Port" -Value "4900"
Set-ItemProperty -Path "HKCU:\SOFTWARE\ODBC\ODBC.INI\remote-db" -Name "Database" -Value "mydb"
```

Then connect:
```
mcp__hfsql__hfsql_connect(id: "remote", dsn: "remote-db", uid: "admin", pwd: "admin")
```
