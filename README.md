# TechLink Claude Plugin

Extends Claude with skills and connectors tailored for TechLink Services — giving every team member (operations, accounting, sales, marketing, recruiting, executives, and developers) the ability to access the SIMPL portal, query company data, and handle incidents without switching between tools.

## Skills

### Query the database
Ask natural-language questions about company data — work orders, clients, invoices, payments, technicians, and more. No SQL knowledge needed.

Example prompts:
- "How many open work orders do we have right now?"
- "Show me all unpaid invoices older than 30 days."
- "Which technicians have the lowest on-time rate this month?"

### Schema explorer
Understand how the company database is structured — what tables exist, what fields they contain, and how they relate to each other.

Example prompts:
- "What tables are in the database?"
- "What fields are available on the work order table?"
- "What data do we track about clients?"

### Portal resources
Look up and review resources in the SIMPL portal — work orders, projects, clients, technicians, invoices, and scheduling.

Example prompts:
- "Find work order #4821."
- "Show me all open jobs for Acme Corp."
- "What's the outstanding balance for this client?"
- "Which technicians are scheduled tomorrow?"

### Incident runbook
Get step-by-step guidance through operational incidents — technician no-shows, mis-assigned jobs, billing errors, disputed charges, and client deliverable problems.

Example prompts:
- "A technician didn't show up for a job."
- "A client is disputing an invoice."
- "We sent the wrong deliverable to a client."

## Connectors

This plugin requires two self-hosted MCP servers (dev-portal and dev-db). After installing the plugin, you must update the MCP server URLs in the plugin's `.mcp.json` file before the connectors will work.

| Server | Placeholder URL to replace |
|--------|---------------------------|
| `dev-portal` | `https://replace-with-your-dev-portal-mcp-url/mcp` |
| `dev-db` | `https://replace-with-your-dev-db-mcp-url/mcp` |

Contact your system administrator or the developer who set up these servers for the correct URLs.

## Setup

1. Install the plugin in Cowork by double-clicking the `.plugin` file.
2. Open the plugin's `.mcp.json` file and replace the two placeholder URLs with your actual MCP server endpoints.
3. Restart Cowork to activate the MCP server connections.
4. The four skills will be available immediately in any Cowork session.
