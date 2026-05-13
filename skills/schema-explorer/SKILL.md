---
name: schema-explorer
description: >
  Use this skill when the user wants to understand the structure of the company database —
  what data exists, how it's organized, or what fields are available. Trigger phrases include:
  "what tables are in the database", "describe the schema", "what fields does X have",
  "what data do we store about", "how is the database structured", "what columns are in",
  "show me the data model", "what information is tracked for", or any question about
  understanding the database layout rather than querying specific records.
metadata:
  version: "0.1.0"
---

When this skill triggers, help the user understand what data the company database contains and how it's structured, using the simpl-db connector.

## Core behavior

1. Use the simpl-db connector to list available tables or describe specific tables as requested.
2. Present schema information in plain language — label fields with plain descriptions, not just raw column names.
3. Group related fields together (e.g., contact info, financial fields, status fields) to make large tables easier to scan.
4. Offer natural next steps: "Want me to pull some example records from this table?" or "I can run a query using any of these fields."

## Translating technical names

Column names are often abbreviated or technical. Interpret and describe them in plain language:
- `wo_id` → Work Order ID
- `tech_id` → Technician ID
- `inv_amt` → Invoice Amount
- `sched_dt` → Scheduled Date

Always prioritize clarity over technical precision for non-developer audiences.

## Scope

This skill explains structure — it does not modify the schema or expose internal database credentials or connection details. If the user asks about a table or field the simpl-db connector does not expose, explain that it isn't available through this connection.
