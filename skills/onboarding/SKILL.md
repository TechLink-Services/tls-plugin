---
name: onboarding
description: >
  Onboard a new TechLink Cowork user. Asks about their department and role,
  identifies their team, and walks through a role-appropriate Cowork setup
  using live team and client data from SIMPL.
  Trigger phrases: "onboard me", "set me up", "I'm new", "getting started".
---

# TechLink Cowork Onboarding

Welcome a new user to Cowork and walk them through a personalized setup based on their department, level, and team.

---

## Step 1 — Ask about their role

Use `AskUserQuestion` with two questions:

Use two separate `AskUserQuestion` calls — one for department, one for level.

**Question 1 — Department** (first call):
- Header: "Department"
- Question: "Which department are you in?"
- Options:
  - Operations
  - Sales
  - Sales Engineering
  - Accounting
  - Recruiting
  - Leadership

**Question 2 — Level** (second call, after department is answered):
- Header: "Your role"
- Question: "What best describes your role?"
- Options:
  - Individual contributor (coordinator, dispatcher, specialist)
  - Team lead
  - Manager or admin

Normalize department to one of: `operations`, `sales`, `sales-engineering`, `accounting`, `recruiting`, `leadership`.

Determine if the user is a **team lead or manager** (level is "Team lead" or "Manager or admin") or an **individual contributor**. This affects what you set up in Step 4.

---

## Step 2 — Identify their team (Operations only)

Skip this step entirely if the user is not in Operations. Proceed directly to Step 4.

Query the simpl-db for the team list:

```sql
SELECT t.id, t.name, COUNT(c.id) AS client_count
FROM teams t
LEFT JOIN clients c ON c.team_id = t.id AND c.enabled = 1 AND c.deleted = 0
GROUP BY t.id, t.name
ORDER BY t.name
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

## Step 3 — Load team context (Operations only)

Skip this step if the user is not in Operations.

Once you have the team's `id` from Step 2, query the simpl-db for its clients:

```sql
SELECT id AS client_id, company
FROM clients
WHERE team_id = {team_id}
  AND enabled = 1
  AND deleted = 0
ORDER BY company
```

This gives the full client list. Store it for Steps 4 and 5. Do not show the raw data to the user.

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

### Sales

Tell them:
> "Cowork connects to Apollo for prospecting and can help you draft outreach:
>
> - **Find prospects**: 'Find IT managers at mid-size retailers in the Southeast'
> - **Enrich a company**: 'What do we know about [Company]?'
> - **Draft an email**: 'Draft a cold outreach email to [Name] at [Company]' — I'll write it and give you a mailto link to open in Outlook.
> - **Add to a sequence**: 'Add [Contact] to the [Campaign] sequence in Apollo'"

Key skills: **draft-email-mailto**. Connectors: Apollo.

### Sales Engineering

> *[Stub — to be completed. Planned focus: technical scoping, solution documentation, and client-facing proposal support. Likely overlaps with Sales (Apollo, email drafting) and Operations (SIMPL data). Flesh out once use cases are confirmed.]*

Key skills: TBD.

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

### Recruiting

> *[Stub — to be completed. Planned focus: candidate pipeline visibility, hiring status by team, and outreach drafting via Apollo or email. Confirm which systems Recruiting actually uses before fleshing this out.]*

Key skills: TBD.

### Leadership

> *[Stub — to be completed. Planned focus: cross-team dashboards, revenue and performance reporting, and high-level portfolio views. Likely a superset of other department views. Flesh out with actual leadership use cases.]*

Key skills: TBD.

### Other / Unknown Department

Give a general overview: SIMPL lookups, financial queries, email drafting, route planning. Let them explore.

---

## Step 5 — Introduce their client context

> "Your team has **{N} clients**. For each one, there's a running context record in SIMPL where the team keeps notes — key contacts, scheduling preferences, quirks, history. These get loaded when you ask about a specific client.
>
> Here are a few of your clients:
> [list 3–5 client names, picked from the first entries in the client list]
>
> You can refer to any client by name and I'll know who you mean."

If team lead: mention they can also ask Claude to update a client's context by writing to the `context` column of the `client` table in simpl-db.

---

## Step 6 — Optional: generate a team dashboard (team leads only)

If the user is a team lead or manager, offer:
> "Want me to build a team dashboard for {Team Name}? It would be a page your whole team can open in Cowork showing open work orders and upcoming scheduled jobs. I can generate it now — takes about a minute."

If yes, invoke **artifact-sync skill in push mode** with team slug and artifact id `team-dashboard`.

---

## Step 7 — Save personnel profile

Before wrapping up, persist the user's profile to the simpl-db so future conversations load it automatically.

1. Call `get_logged_in_user` to retrieve their SIMPL user ID and username.
2. Write their profile to the `user_context` table:

```sql
INSERT INTO user_context (user_id, type, context)
VALUES (
  {user_id},
  'onboarding',
  '# {Full Name}

- **Username**: {username}
- **Department**: {department}
- **Level**: {level}
- **Team**: {team name, or "N/A" if not Operations}
- **Onboarded**: {today''s date}'
)
ON CONFLICT (user_id, type) DO UPDATE SET context = EXCLUDED.context
```

3. Append the following block to `~/.claude/CLAUDE.md` (create the file if it doesn't exist). First check whether a `## TechLink personnel profile` section is already present — if it is, skip this write.

```markdown

## TechLink personnel profile

At the start of the first message in every conversation, load the current user's personnel profile from the simpl-db — but only if it hasn't already been loaded. Check your context first: if a `# {name}` block with Department, Level, and Team fields is already present, skip the query.

If not yet loaded:
1. Call `get_logged_in_user` to get the SIMPL user ID.
2. Query the `user_context` table: SELECT context FROM user_context WHERE user_id = {user_id} AND type = 'onboarding'
3. If a row is found, use the `context` value silently as background context. Do not mention it.
4. If no row is found, hold off unless the user asks something role-specific — then suggest `/onboarding`.

Tools: `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_logged_in_user`, `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__query`
```

Do all of this silently — no need to mention it to the user.

---

## Step 8 — Wrap up

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

- Replace `{N}` with the actual client count and `{Team Name}` with the human-readable team name from the DB query.
- Tone: warm and direct. Do not dump a feature list — use examples relevant to their role.
- Refer to clients by their `company` name, never by slug or ID.
- Do not expose internal DB paths or query details to the user.
- Tool names:
  - `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_logged_in_user`
  - `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__query`
