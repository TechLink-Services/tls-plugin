---
name: route-jobs
description: >
  Use this skill when the user asks about unassigned or unscheduled jobs for a specific
  client, especially when they want to see where those jobs are located or grouped
  geographically. Trigger phrases include: "unscheduled jobs for", "unassigned jobs for",
  "what jobs haven't been scheduled for", "jobs with no tech for", "open dispatch queue
  for", "what's not assigned for", "show me jobs that need scheduling for", or any request
  combining a client name with the idea of jobs that are pending assignment or scheduling.
  Also triggers for follow-up actions on surfaced clusters: "run auto-dispatch", "find
  installers near", "build a route for", "schedule these in", "create route sheets", or
  any request to plan, dispatch, or generate Excel documents from an unscheduled job list.
  Always use this skill (rather than portal-resources or query-db) when the user wants
  unassigned/unscheduled jobs grouped or sorted by location, or wants to take action on
  those clusters.
metadata:
  version: "0.1.0"
---

When this skill triggers, retrieve unassigned or unscheduled work orders for a specific
client, present them grouped by geographic proximity, and — if the user wants to act on
the results — run auto-dispatch, build routes, schedule across a target month, and
generate Excel route sheets for installer-group approval.

---

# Part 1 — Unscheduled Jobs by Location

## Step 1 — Confirm the client

If the user named a client, proceed. If not, ask: "Which client would you like to check?"

Look up the client ID:
```sql
SELECT client_id, company
FROM client
WHERE company LIKE '%<name>%'
```
Use `client_id` and `company` — those are the correct column names. The `client` table has
no `id` or `name` column.

If multiple clients match, list them and ask the user to confirm which one.

## Step 2 — Fetch unassigned/unscheduled work orders

Use the `workorders_unassigned_or_unscheduled` tool with:
- `client_id`: the ID found in Step 1
- `team_id`: 0 (all teams)

This returns all work orders where statcode ≤ 60 and either no installer is assigned or
no install start date is set.

Note the `id` of every returned work order — you'll need them in Step 3.

## Steps 3–5 — Enrich with location data, cluster, and present results

Delegate to the `geo-clusterer` agent, passing the list of WO IDs from Step 2. It will
fetch address and coordinate data, group jobs into geographic clusters using the Haversine
formula, and present the results grouped by region. Wait for it to finish before
proceeding to Step 6.

---

# Part 2 — Route Planning & Excel Schedule Generator

Triggered when the user selects one or more clusters and wants to run auto-dispatch,
build routes, schedule work across a month, and produce Excel sheets for installer
approval.

## Step 6 — Identify Target Regions

Identify clusters from part 1 with more than 5 jobs that are good opportunities for 
dispatching to a single Installer group to travel between.

## Step 7 — Select a Representative Work Order Per Route

For each route, pick one **centrally located work order** to anchor the auto-dispatch
search. Good anchors are jobs near the geographic centroid of the cluster (e.g. a
San Francisco job for the Bay Area, a Brooklyn job for NY/NJ, a Boca Raton job for
South Florida).

## Step 8 — Run Auto-Dispatch

Call `auto_dispatch` in **parallel** for all representative work orders simultaneously.
Use a `dispatch_dist` of 75 miles to ensure adequate installer coverage.

```
auto_dispatch(workorder_id=<anchor_wo>, dispatch_dist=75)
```

Collect the full list of installer values returned for each region.


## Step 9 — Build Day-by-Day Routes

For each region, organize all work orders into a logical travel sequence:

- **Sort geographically** — group stops to minimize backtracking (north-to-south or
  loop patterns)
- **Cluster nearby stops on the same day** — aim for 1–2 stops per day depending on
  job complexity (installs vs. surveys)
- **Note jobs with complex scopes** (large device counts, service calls) as candidates for dedicated time blocks

For large regions (e.g. Bay Area with 25+ jobs), split into named sub-sweeps:
- Sacramento/Foothills sweep
- East Bay sweep
- Peninsula/South Bay sweep
- Coastal/Central Valley sweep

## Step 10 — Schedule Across the Target Month

Ask the user for a target month if not already stated. Distribute routes across available
business days so that **work volume is roughly equal across each week**. Routes can run
**simultaneously** — do not schedule them sequentially unless the user requests it.

Key rules:
- Avoid national holidays (e.g. Memorial Day = last Monday in May)
- Recommend arrive-day travel the evening before the first work day
- Recommend depart-day travel the morning after the last work day
- For large regions needing multiple weeks, split into Week 2 and Week 3 blocks

**Target distribution:** divide total jobs by number of available weeks; aim for
±2 jobs variance per week across all active routes combined.

**Example for 58 jobs over 3 weeks (12 business days):**
- Week 1: ~17 jobs (2 smaller routes run simultaneously)
- Week 2: ~21 jobs (1 large route Days 1–5 + 1 medium route Days 1–2, simultaneously)
- Week 3: ~20 jobs (1 large route Days 6–7 + 1 medium route Days 3–4, simultaneously)

## Step 11 — Confirm Excel Generation

Present the completed route plan summary to the user, then ask:

> "Would you like me to generate Excel route sheets for these routes?"

If yes, proceed to Step 12. If no, stop here.

## Step 12 — Generate Excel Route Sheets

Delegate to the `excel-route-generator` agent, passing all route data: client name and
slug, and for each route — its name, region slug, installer pool, day-by-day job schedule,
scheduled date range with arrive/depart notes, and region notes. The agent generates and
validates all `.xlsx` files in parallel and delivers them as `computer://` links grouped
by week.