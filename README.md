# hfsql-skill

Claude Code skill for interacting with **HFSQL databases** (PC SOFT / WinDev).

Works with the [hfsql-mcp](https://github.com/YOUR_USER/hfsql-mcp) server.

## Installation

Copy `hfsql.md` into your Claude Code skills directory:

```
~/.claude/skills/hfsql.md
```

Or for project-level:
```
.claude/skills/hfsql.md
```

## What it does

This skill teaches Claude Code how to:

- Connect to HFSQL databases via ODBC DSN
- Explore schemas (tables, columns, types, row counts)
- Query data with proper HFSQL SQL syntax
- Create, update, and delete records safely
- Handle HFSQL quirks: calculated fields, date formats (`YYYYMMDD`), French-accented identifiers
- Create ERP documents (invoices, credit notes, delivery notes) with correct reference numbering
- Respect fiscal year locks and document chains

## Prerequisites

- The `hfsql-mcp` MCP server must be installed and configured in `.mcp.json`
- An HFSQL ODBC DSN must be configured in Windows ODBC Manager
- HFSQL ODBC driver installed (PC SOFT)

## License

MIT
