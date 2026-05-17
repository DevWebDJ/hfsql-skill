# hfsql-skill

Claude Code skill for interacting with **HFSQL databases** (PC SOFT / WinDev).

Works with the [hfsql-mcp](https://www.npmjs.com/package/hfsql-mcp) server.

## Installation

### 1. Install the MCP server

```bash
npm install -g hfsql-mcp
```

### 2. Add the MCP server to Claude Code

Run this command in your terminal:

```bash
claude mcp add hfsql -- npx hfsql-mcp
```

Or manually add to your `~/.mcp.json`:

```json
{
  "mcpServers": {
    "hfsql": {
      "command": "npx",
      "args": ["hfsql-mcp"]
    }
  }
}
```

### 3. Install the skill

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

- **Windows** with PowerShell 7+ (`pwsh`)
- **Node.js** 18+
- **HFSQL ODBC driver** installed (PC SOFT)
- An **ODBC DSN** configured in Windows ODBC Manager pointing to your HFSQL server

## License

MIT
