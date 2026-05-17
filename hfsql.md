---
name: hfsql
description: >
  Interact with HFSQL databases (PC SOFT / WinDev) via the hfsql MCP server.
  Handles connections, schema exploration, queries, inserts, updates, and deletes.
  Automatically manages HFSQL quirks: calculated fields, date formats, French-accented identifiers, and the absence of INFORMATION_SCHEMA.
trigger: >
  When the user asks to query, explore, modify, or manage an HFSQL database,
  or references tables, articles, mouvements, tiers, factures, or any ERP-related data
  in a WinDev/HFSQL context. Also triggered by mentions of DSN, ODBC + HFSQL,
  or the hfsql MCP tools.
---

# HFSQL Database Skill

You have access to an HFSQL MCP server (`hfsql`) that provides 8 tools for full database interaction. All tools communicate via PowerShell ODBC and write results as UTF-8 JSON via temp files — no console encoding issues.

## Available Tools

| Tool | Purpose |
|------|---------|
| `hfsql_connect` | Open a connection via ODBC DSN |
| `hfsql_disconnect` | Close a connection |
| `hfsql_list_tables` | List all tables (with optional row counts) |
| `hfsql_describe_table` | Get column schema for a table |
| `hfsql_query` | Run SELECT queries → JSON results |
| `hfsql_execute` | Run INSERT / UPDATE / DELETE |
| `hfsql_insert` | Smart insert from JSON object (auto-handles calculated fields + dates) |
| `hfsql_schema_summary` | Full schema dump (all tables, all columns, types, row counts) |

## Connection Workflow

Always connect first before any operation:

```
hfsql_connect(id: "<name>", dsn: "<odbc_dsn>", uid: "<user>", pwd: "<password>")
```

- `id` is a friendly name you choose (e.g. "kbpro") — reuse it in all subsequent calls
- `dsn` is the ODBC Data Source Name configured in Windows ODBC Manager
- Default credentials: uid="admin", pwd="admin"

## HFSQL-Specific Rules

### 1. No SELECT without FROM
HFSQL has no DUAL table. `SELECT 1` will fail. Always query from a real table:
```sql
SELECT COUNT(*) FROM [Articles]
```

### 2. Table and column names with accents
Use square brackets for names with French accents:
```sql
SELECT [Libellé], [Référence] FROM [Règlement]
```

### 3. Date format
HFSQL expects dates as `YYYYMMDD` (no dashes, no slashes):
```sql
WHERE Date = '20250517'
```
The `hfsql_insert` tool auto-converts `YYYY-MM-DD` → `YYYYMMDD`.

### 4. Calculated fields
Some columns (like `Mois` on `Mouvement`) are calculated by HFSQL and reject INSERT/UPDATE. The `hfsql_insert` tool detects these automatically via `IsReadOnly`/`IsAutoIncrement` schema flags and skips them. For raw `hfsql_execute`, you must exclude them yourself.

### 5. Type column for document types
The `Mouvement` table stores ALL document types (invoices, delivery notes, purchase orders, credit notes, etc.) differentiated by the `Type` column. Common types:

| Type | Prefix | Document |
|------|--------|----------|
| 11 | CF | Bon de commande fournisseur |
| 12 | BR | Bon de réception |
| 13 | BRF | Bon de retour fournisseur |
| 14 | FA | Facture d'achat |
| 15 | AA | Avoir sur achat |
| 21 | DV | Devis |
| 22 | CC | Bon de commande client |
| 23 | BL | Bon de livraison |
| 24 | BRC | Bon de retour client |
| 25 | FV | Facture de vente |
| 26 | AV | Avoir sur vente |
| 33 | BSor | Bon de sortie |
| 34 | BRei | Bon de réintégration |
| 35 | BT | Bon de transfert |

### 6. Reference format
Documents follow the pattern: `{PREFIX}{YY}-{NUMBER:05d}`
Example: `FV25-00020` = Facture de vente n°20 de 2025.

### 7. ID columns
Each table has an auto-increment primary key named `ID{TableName}` (e.g. `IDMouvement`, `IDArticle`, `IDLigne_MT`).

## Common Query Patterns

### Count documents by type for a year
```sql
SELECT Type, COUNT(*) AS Total FROM [Mouvement] WHERE Année = 2025 GROUP BY Type
```

### Get latest invoice
```sql
SELECT * FROM [Mouvement] WHERE Type = 25 AND Année = 2025 ORDER BY Numéro DESC
```

### Get line items for a document
```sql
SELECT * FROM [Ligne_MT] WHERE IDMouvement = 723
```

### Get client/supplier info
```sql
SELECT * FROM [Tiers] WHERE IdTiers = 30
```

### Get article details
```sql
SELECT IDArticle, Reference, Libellé, Prix_Achat_HT, Prix_Vente_HT FROM [Articles]
```

### Check fiscal year status
```sql
SELECT Année, Verrou FROM [Exercice]
```

## Creating Documents

When creating a new document (invoice, credit note, etc.):

1. **Query `Type_Pièce`** to get the correct `Type` code and prefix
2. **Find the next number**: `SELECT MAX(Numéro) FROM [Mouvement] WHERE Type = X AND Année = Y`
3. **Build the reference**: `{PREFIX}{YY}-{NEXT_NUMBER:05d}`
4. **Insert the header** in `Mouvement` using `hfsql_insert` — skip `Mois` (calculated)
5. **Insert lines** in `Ligne_MT` referencing the new `IDMouvement`
6. **Verify** with `hfsql_query`

### Important fields for Mouvement header
- `Année`, `Numéro`, `Référence`, `Date` (YYYYMMDD), `Type`, `ST`, `Statut`
- `IdTiers` (client/supplier ID)
- `HT`, `TVA`, `TTC`, `Net_HT`, `Solde`
- `Saisi_le`, `Saisi_par`, `Modif_Le`, `Modif_Par`
- `IdSociété` (usually 1)

### Important fields for Ligne_MT
- `IDMouvement` (FK to header), `Type` (same as header), `IDArticle`
- `Qte`, `Prix_HT`, `Prix_TTC`, `Résultat_HT`, `Résultat_TTC`
- `Ordre` (line order), `IdSociété`

## Safety Guidelines

- **Always confirm** before INSERT/UPDATE/DELETE operations on production data
- **Check fiscal year lock** (`Verrou` in `Exercice`) before modifying transactions
- **Verify references** don't already exist before inserting
- **Use `hfsql_query` to verify** after any modification
- **Never modify `Paramètres`** (886-column system config) without explicit user request
- **Watch for cascading effects**: documents link via `MT_Associer`, payments via `Règlement`/`Ligne_RG`

## Error Handling

If a query fails with "rubrique calculée" (calculated field), the column is read-only — remove it from your INSERT/UPDATE. The `hfsql_insert` tool handles this automatically.

If a query fails with "Fichier XXX inconnu", the table name is wrong — check with `hfsql_list_tables`.

If dates appear garbled (like `07/20/0005`), the format was wrong — use `YYYYMMDD` strictly.
