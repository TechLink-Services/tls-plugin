---
name: geo-clusterer
description: >
  Fetch address and coordinate data for a list of work order IDs, then group them into
  geographic clusters using the Haversine formula. Invoke from route-jobs after fetching
  unassigned work orders — pass the list of WO IDs and this agent returns labeled clusters
  ready for route planning.
model: sonnet
effort: medium
maxTurns: 10
---

Given a list of work order IDs, fetch location data and group jobs into geographic clusters.

## Step 1 — Fetch location data

Run this query, substituting the provided WO IDs. Use this exact join pattern — do not deviate. The naive approaches (`site.addr_id`, direct column guesses) do not work.

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
WHERE wo.id IN (<comma-separated WO ids>)
```

**Critical schema notes:**
- `workorder.id` — primary key is `id`, not `workorder_id`
- `site.id` — use `id`, not `site_id`
- Address-to-site join: `address.entity_id = site.id AND address.address_type = 'site'` — `site.addr_id` is always 0 and must NOT be used
- `address` columns: `address1`, `city`, `state` (int FK), `zip`, `lat`, `lng`
- `state` table: join on `address.state = state.id`, use `state.abbr` for abbreviation
- `site.company` — the store name; `site.siteid` — the client's internal store number

## Step 2 — Cluster by geographic proximity

Use the Haversine formula to calculate distances in miles between every pair of sites:

```
d = 3958.8 × 2 × arcsin(√(
      sin²((lat2−lat1)/2) +
      cos(lat1) × cos(lat2) × sin²((lng2−lng1)/2)
    ))
```
(all angles in radians)

Clustering approach: pick the job with the most neighbors within 75 miles as the cluster anchor, assign all within-range jobs to that cluster, then repeat for remaining unassigned jobs. Label each cluster by its anchor's city and state (e.g., "Minnetonka, MN area").

Jobs with no coordinates (lat = 0 and lng = 0) go in a separate "No location data" section.

## Step 3 — Present results

For each geographic cluster:
- Header: **[City, State] area — N job(s)**
- Table: WO #, Summary, Site Name, Site ID, Address, Status

Statcode reference:

| statcode | Label |
|---|---|
| 22 | Dispatched (no tech) |
| 26 | Open (unassigned) |
| 40 | Assigned |
| 50 | Accepted |
| 55 | Pending |
| 60 | Scheduled (no date) |

At the bottom, note the total count and flag any statcode 50 jobs with a scheduled date that has already passed — these had a technician but no confirmed install date and may be overdue.
