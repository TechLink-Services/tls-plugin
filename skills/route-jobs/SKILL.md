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

## Step 3 — Enrich with location data

Fetch address and coordinates for all returned work orders in a single query. Use this
exact join pattern — **do not deviate**. The naive approaches (`site.addr_id`, direct
column guesses) do not work; this is the query that does:

```sql
SELECT
  wo.id,
  wo.summary,
  wo.statcode,
  s.company   AS site_name,
  s.siteid,
  a.address1,
  a.city,
  st.abbr     AS state,
  a.zip,
  a.lat,
  a.lng
FROM workorder wo
LEFT JOIN site    s  ON wo.site_id     = s.id
LEFT JOIN address a  ON a.entity_id   = s.id
                     AND a.address_type = 'site'
LEFT JOIN state   st ON a.state       = st.id
WHERE wo.id IN (<comma-separated list of WO ids>)
```

**Critical schema notes:**
- `workorder.id` — the primary key is `id`, not `workorder_id`
- `site.id` — use `id`, not `site_id`
- Address-to-site join: `address.entity_id = site.id AND address.address_type = 'site'`
  — `site.addr_id` is always 0 and must NOT be used
- `address` columns: `address1`, `city`, `state` (int FK), `zip`, `lat`, `lng`
- `state` table: join on `address.state = state.id`, use `state.abbr` for abbreviation
- `site.company` — the store name; `site.siteid` — the client's internal store number

## Step 4 — Group by geographic proximity

Use the Haversine formula to calculate the distance in miles between every pair of job
sites, then cluster jobs within roughly 75 miles of each other into the same group.

**Haversine formula (miles):**
```
d = 3958.8 × 2 × arcsin(√(
      sin²((lat2−lat1)/2) +
      cos(lat1) × cos(lat2) × sin²((lng2−lng1)/2)
    ))
```
(all angles in radians)

Clustering approach: pick the job with the most neighbors within 75 miles as the cluster
anchor, assign all within-range jobs to that cluster, then repeat for remaining unassigned
jobs. Label each cluster by its anchor's city and state (e.g., "Minnetonka, MN area").

If any work orders have no coordinates (lat = 0 and lng = 0), list them in a separate
"No location data" section at the bottom rather than silently omitting them.

## Step 5 — Present results

For each geographic cluster, show:
- A header: **[City, State] area — N job(s)**
- A table with columns: WO #, Summary, Site Name, Site ID, Address, Status

Use the statcode reference to translate numeric statcodes to human-readable labels:

| statcode | Label |
|---|---|
| 22 | Dispatched (no tech) |
| 26 | Open (unassigned) |
| 40 | Assigned |
| 50 | Accepted |
| 55 | Pending |
| 60 | Scheduled (no date) |

At the bottom, note the total count and flag any jobs where statcode is 50 with a
scheduled date that has already passed — these had a technician but no confirmed install
date and may be overdue.

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

## Step 11 — Generate Excel Route Sheets

Create one `.xlsx` file per route using `openpyxl`. Each file contains two sheets.

### Sheet 1: Route Summary

| Field | Content |
|---|---|
| Title banner | Route name, navy header (#1F3864), white Arial bold 14pt |
| Client | Client name |
| Route | Route name |
| Total Jobs | Count |
| Scheduled Dates | Date range + arrive/depart notes |
| Region Notes | Parallelization guidance, tech recommendations |
| Installer Pool table | Installer ID, Company/Contact, City, State, Notes |

Installer pool header: blue (#2E75B6), column headers navy, alternating white/#F2F2F2 rows.

### Sheet 2: Job Schedule

Columns: `Stop | WO # | Site Name | Site ID | Address | City | State | Work Summary | Status`

- One **day-header row** per day (blue #2E75B6, white bold text, merged across all columns)
- Column header row beneath each day header (navy, white bold)
- Data rows alternate white/#F2F2F2, overridden by status color:
  - Open (new): white
  - Open (unassigned): #FFF2CC (yellow)
  - Dispatched (no tech): #FCE4D6 (orange)
  - Pending: #F8CBAD (dark orange)
- Wrap text on all cells, row height 30
- Status legend at the bottom of the sheet
- Freeze top row, hide gridlines

### File naming convention
```
<Client_Slug>_<RegionSlug>_Route.xlsx
```
Examples: `Constant_Media_NYNJ_Route.xlsx`, `Constant_Media_CA_BayArea_Route.xlsx`

### Validation
After saving, run `scripts/recalc.py` on each file and confirm
`"status": "success"` with `"total_errors": 0` before delivering.

## Output

Present all files as `computer://` links grouped by week, followed by a summary table:

| Week | Dates | Routes Active | Jobs |
|---|---|---|---|
| Week 1 | … | … | … |
| Week 2 | … | … | … |
| Week 3 | … | … | … |