---
name: onboarding
description: >
  Onboard a new TechLink Cowork user. Takes --department and --level arguments,
  asks which team the user is joining, and walks through a role-appropriate Cowork
  setup using live team and client data from S3 and SIMPL.
  Trigger phrases: "onboard me", "set me up", "I'm new", "getting started",
  or any invocation with --department and/or --level flags.
---

# TechLink Cowork Onboarding

Welcome a new user to Cowork and walk them through a personalized setup based on their department, level, and team.

---

## Step 1 — Parse arguments

The skill may be invoked with arguments:
- `--department` (e.g. `operations`, `project management`, `accounting`, `sales`)
- `--level` (e.g. `project coordinator`, `dispatcher`, `team lead`, `manager`, `admin`)

If either argument is missing, ask before proceeding.

Normalize department to one of: `operations`, `project-management`, `accounting`, `sales`, `other`.

Determine if the user is a **team lead or manager** (level contains "lead", "manager", or "admin") or an **individual contributor**. This affects what you set up in Step 4.

---

## Step 2 — Identify their team

Read the team list from S3:

```
get_object(key: "teams/index.json")
```

The response contains a list of teams with `name`, `slug`, and `client_count`. Present the team names clearly and ask the user which team they're joining.

Example output:

```
Which team will you be on? Here are the active teams:

  B Squared (9 clients)
  Datacom (22 clients)
  DD Coates (1 client)
  Digital Technology (8 clients)
  Enterprise Group (22 clients)
  Infrastructure Group (22 clients)
  McD Coates (3 clients)
  Retail Services (23 clients)
```

Wait for the user to select a team. Match their answer to a slug (case-insensitive, partial match okay).

---

## Step 3 — Load team context

Once you have their team slug, read:

```
get_object(key: "teams/{slug}/index.json")
```

This gives the full client list (`client_id`, `company`, `slug`). Store it for Steps 4 and 5. Do not show the raw data to the user.

---

## Step 4 — Department-specific orientation

Deliver a concise, role-specific orientation. Tailor based on normalized department and whether they are a team lead/manager or IC.

### Operations

Tell them:
> "Most of what you'll do in Cowork connects to your work orders and scheduling. Here's what you can ask me:
>
> - **Look up a work order**: 'What's the status of WO 12345?' — I'll pull details, notes, tech assignment, and tracking.
> - **Dispatch unscheduled jobs**: 'Show me unassigned jobs for [Client]' — I'll group them by geography and help you build routes.
> - **Mesmerize confirmations**: 'Send a confirmation for WO 12345' — I'll pull the WO, site address, tracking info, and draft the email ready to send.
> - **Scheduling checks**: 'What's scheduled this week for [Client]?' — I'll pull the calendar.
>
> Your team manages **{N} clients**. I know who they are, so you can refer to them by name."

If team lead, additionally:
> "As a team lead, you can also ask me to build a team dashboard — a snapshot of open WOs and upcoming jobs across all your clients that your whole team can see."

Key skills: **portal-resources**, **route-jobs**, **mesmerize-confirm**.

### Project Management

Tell them:
> "Your work lives at the intersection of clients and projects. Here's what Cowork can do for you:
>
> - **Client status**: 'What's open for [Client]?' — I'll pull recent work orders, open items, and project status from SIMPL.
> - **Project overview**: 'Show me all projects for [Client]' — I'll list them with managers and statuses.
> - **Notes and context**: Each of your clients has a context file where the team keeps running notes — preferences, contacts, patterns. Ask me to pull it up or add to it.
> - **Reports**: 'Give me a summary of uninvoiced work for [Client]' — I can query the database for financial and status snapshots.
>
> Your team handles **{N} clients**. I can reference any of them by name."

If team lead:
> "You can ask me to generate a team dashboard — a formatted view of open work orders and upcoming jobs across your portfolio, pushed to your whole team."

Key skills: **portal-resources**, **query-db**, **artifact-sync**.

### Accounting

Tell them:
> "Cowork connects directly to SIMPL's financial data. You can ask me:
>
> - **Aging report**: 'Show me outstanding invoices older than 30 days'
> - **Client balance**: 'What does [Client] owe?'
> - **Uninvoiced work**: 'What work orders need invoicing for [Client]?'
> - **Revenue by month**: 'What did we earn from [Client] in Q1?'
>
> I work from live SIMPL data, so the numbers are always current."

Key skills: **query-db**, **portal-resources**.

### Sales

Tell them:
> "Cowork connects to Apollo for prospecting and can help you draft outreach:
>
> - **Find prospects**: 'Find IT managers at mid-size retailers in the Southeast'
> - **Enrich a company**: 'What do we know about [Company]?'
> - **Draft an email**: 'Draft a cold outreach email to [Name] at [Company]' — I'll write it and give you a mailto link to open in Outlook.
> - **Add to a sequence**: 'Add [Contact] to the [Campaign] sequence in Apollo'"

Key skills: **draft-email-mailto**. Connectors: Apollo.

### Other / Unknown Department

Give a general overview: SIMPL lookups, financial queries, email drafting, route planning. Let them explore.

---

## Step 5 — Introduce their client context

> "Your team has **{N} clients**. For each one, there's a running context file where the team keeps notes — key contacts, scheduling preferences, quirks, history. These get loaded when you ask about a specific client.
>
> Here are a few of your clients:
> [list 3–5 client names, picked from the first entries in the index]
>
> You can refer to any client by name and I'll know who you mean."

If team lead: mention they can also ask Claude to update client context files with new notes.

---

## Step 6 — Optional: generate a team dashboard (team leads only)

If the user is a team lead or manager, offer:
> "Want me to build a team dashboard for {Team Name}? It would be a page your whole team can open in Cowork showing open work orders and upcoming scheduled jobs. I can generate it now — takes about a minute."

If yes, invoke **artifact-sync skill in push mode** with team slug and artifact id `team-dashboard`.

---

## Step 7 — Wrap up

> "You're all set. Quick cheat sheet:
>
> - Ask me about any work order, client, or project by name
> - 'Look up WO [number]' to pull a work order
> - 'Unassigned jobs for [Client]' to see what needs scheduling
> - 'Show context for [Client]' to see the team's running notes
> - 'Update context for [Client]' to add something to those notes
>
> Your team is **{Team Name}** with {N} clients. Anything you want to try right now?"

---

## Implementation notes

- Replace `{N}` with the actual client count and `{Team Name}` with the human-readable name from the S3 index.
- Tone: warm and direct. Do not dump a feature list — use examples relevant to their role.
- Refer to clients by their `company` name, never by slug or ID.
- Do not expose S3 keys or internal paths to the user.
- S3 tool names:
  - `mcp__3e8ed9ed-43e2-4c2d-aafe-fba01ed5d04f__get_object`
  - `mcp__3e8ed9ed-43e2-4c2d-aafe-fba01ed5d04f__list_objects`
