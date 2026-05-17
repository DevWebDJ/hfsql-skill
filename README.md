# hfsql-skill

Claude Code skill for interacting with **HFSQL databases** (PC SOFT / WinDev).

Works with the [hfsql-mcp](https://www.npmjs.com/package/hfsql-mcp) server.

## Installation

### 1. Add the MCP server to Claude Code

```bash
claude mcp add hfsql -- npx hfsql-mcp
```

### 2. Install the skill

```bash
claude skill add --from https://github.com/kbdevs/hfsql-skill
```

Or manually: copy `skills/hfsql/SKILL.md` into your Claude Code skills directory.

## What it does

This skill teaches Claude Code how to:

- Connect to HFSQL databases via ODBC DSN (local or remote)
- Explore schemas (tables, columns, types, row counts)
- Query data with proper HFSQL SQL syntax
- Create, update, and delete records safely
- Handle HFSQL quirks: calculated fields, date formats (`YYYYMMDD`), French-accented identifiers
- Create ERP documents (invoices, credit notes, delivery notes) with correct reference numbering
- Respect fiscal year locks and document chains
- Set up ODBC DSN via GUI or PowerShell (for headless/remote servers)

## Prerequisites

- **Windows** with PowerShell 5.1+ (PowerShell 7 preferred but not required — the server auto-detects both)
- **Node.js** 18+
- **HFSQL ODBC driver** installed (PC SOFT — comes with WinDev or standalone)
- An **ODBC DSN** configured in Windows ODBC Data Source Administrator

## License

MIT
