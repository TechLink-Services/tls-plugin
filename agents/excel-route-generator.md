---
name: excel-route-generator
description: >
  Generate Excel route sheets (.xlsx) for installer groups from a completed route plan.
  Invoke from route-jobs after routes are fully planned and scheduled — pass all route
  data and this agent produces one .xlsx file per route, validates each with recalc.py,
  and delivers them as computer:// links. Runs in an isolated worktree.
model: sonnet
effort: medium
maxTurns: 20
isolation: worktree
skills:
  - techlink-claude-plugin:xlsx
---

Generate one `.xlsx` route sheet per route from the provided route plan data. Create all files in parallel — each is independent.

## Input expected

Receive from the calling skill:
- Client name and slug
- List of routes, each containing:
  - Route name and region slug
  - Installer pool (ID, company/contact, city, state, notes)
  - Day-by-day job schedule with stops (WO#, site name, site ID, address, city, state, work summary, status)
  - Scheduled date range + arrive/depart notes
  - Region notes (parallelization guidance, tech recommendations)

## File naming

```
<Client_Slug>_<RegionSlug>_Route.xlsx
```

Examples: `Constant_Media_NYNJ_Route.xlsx`, `Constant_Media_CA_BayArea_Route.xlsx`

## Sheet 1: Route Summary

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

## Sheet 2: Job Schedule

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

## Validation

After saving each file, run `scripts/recalc.py` on it and confirm `"status": "success"` with `"total_errors": 0` before delivering.

## Output

Present all files as `computer://` links grouped by week, followed by a summary table:

| Week | Dates | Routes Active | Jobs |
|---|---|---|---|
