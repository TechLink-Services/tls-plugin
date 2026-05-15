---
name: artifact-sync
description: >
  Generate and deploy a Cowork artifact for a team, or sync all team artifacts from S3 to Cowork.
  Use when a team lead says: "create a dashboard for [team]", "update the [name] artifact",
  "push a [report] to the team", "refresh team artifacts", or "sync artifacts from S3".
  Also used by the daily scheduled sync task.
---

# Artifact Sync

This skill covers two modes that share the same S3 + Cowork artifact infrastructure:

- **Push mode**: A team lead asks Claude to create or update a specific artifact. Claude generates the HTML, uploads to S3, and immediately creates or updates the Cowork artifact.
- **Sync mode**: A scheduled task runs daily. Claude reads all artifact HTML files from S3 and creates or updates the corresponding Cowork artifacts for all teams.

---

## S3 naming conventions

Artifacts live under the `artifacts/` prefix in the `techlink-cowork-artifacts` bucket:

```
artifacts/{team-slug}/{artifact-id}.html
```

Examples:
```
artifacts/retail-services/team-dashboard.html
artifacts/operations/team-dashboard.html
artifacts/retail-services/client-mesmerize.html
```

The Cowork artifact ID is derived from the S3 key by joining the team slug and filename with a hyphen and stripping `.html`:

```
artifacts/retail-services/team-dashboard.html  →  retail-services-team-dashboard
artifacts/retail-services/client-mesmerize.html  →  retail-services-client-mesmerize
```

---

## Push mode — generate and deploy an artifact

Use this when a team lead asks Claude to create or update a specific artifact.

### Step 1 — Clarify intent (if needed)

If the user didn't specify team slug and artifact name, ask for both before proceeding.
- Team slug: one of the slugs from `teams/index.json` in S3 (e.g. `retail-services`, `operations`)
- Artifact type: `team-dashboard`, `client-{client-slug}`, or a custom name

If the user says "update the dashboard for retail services", infer:
- team slug: `retail-services`
- artifact id: `team-dashboard`

### Step 2 — Gather live data, then generate the HTML

Pull the data the artifact needs before writing any HTML. Run data fetches in parallel where possible.

For **team dashboards**:
- Open work orders: `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__team_open_workorders`
- Scheduled jobs this week: `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__workorders_scheduled_in_date_range`
- Team client list: read `teams/{team-slug}/index.json` from S3

For **client pages**:
- Recent work orders: `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__client_recent_workorders`
- Client context: `get_object(key: "teams/{team-slug}/clients/{client-slug}/context.md")`

Generate a complete, self-contained HTML document. Requirements:
- Light mode: include `<style>:root { color-scheme: light }</style>`, use a light background with dark text
- Inline all CSS and JS — no external stylesheets except the three CDN libraries listed below
- Embed the current date as a visible "Generated: YYYY-MM-DD" timestamp in the HTML
- Do NOT use `window.cowork.callMcpTool` in the HTML — data is fetched during generation and baked in. The artifact is a snapshot, not a live dashboard.

Allowed CDN libraries (use exact tags):
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.5.0/dist/chart.umd.js" integrity="sha384-iU8HYtnGQ8Cy4zl7gbNMOhsDTTKX02BTXptVP/vqAWIaTfM7isw76iyZCsjL2eVi" crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/gridjs@5.0.2/dist/gridjs.umd.js" integrity="sha384-/XXDzxe4FsGiAe50i/u9pY/Vy/uX654MHB1xoc1BJNnH1WXHhqHga9g3q5tF4gj7" crossorigin="anonymous"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/gridjs@5.0.2/dist/theme/mermaid.min.css" integrity="sha384-jZvDSsmGB9oGGT/4l9bHXGoAv1OxvG/cFmSo0dZaSqmBgvQTKDBFAMftlXTmMbNW" crossorigin="anonymous">
```

### Step 3 — Write HTML to workspace, then upload to S3 and create/update Cowork artifact

Write the HTML to a temp file, then run S3 upload and Cowork artifact create/update in parallel.

**A. Upload to S3** — call `put_object`:

```
key:         artifacts/{team-slug}/{artifact-id}.html
content:     (the HTML string)
contentType: text/html
```

**B. Create or update Cowork artifact**:
- Call `list_artifacts` to check whether an artifact with this ID already exists.
- Not found → `create_artifact(id: "{team-slug}-{artifact-id}", html_path, description: one-sentence summary)`
- Found → `update_artifact(id, html_path, update_summary: "Refreshed with current data")`

### Step 4 — Confirm

Tell the user the artifact name, its Cowork ID, and the S3 key it was saved to.

---

## Sync mode — pull all S3 artifacts to Cowork

Use this for the daily scheduled task or when the user says "sync artifacts from S3".

### Step 1 — List all artifacts in S3

Call `list_objects` with prefix `artifacts/`. Filter to keys ending in `.html`.

### Step 2 — Cache existing Cowork artifacts

Call `list_artifacts` ONCE and store the result. Do not call it in a loop.

### Step 3 — Sync each file

For each `.html` key:

**A.** Derive Cowork artifact ID: strip `artifacts/` prefix, replace `/` with `-`, remove `.html`.
Example: `artifacts/retail-services/team-dashboard.html` → `retail-services-team-dashboard`

**B.** Read from S3 with `get_object`. If `{ found: false }`, skip and log.

**C.** Write HTML to a temp file in the workspace.

**D.** Create or update using the cached artifact list.

### Step 4 — Report results

```
Synced N artifacts across M teams.
  Created: [list]
  Updated: [list]
  Skipped: [any not found in S3]
```

---

## S3 connector tool names

- `mcp__3e8ed9ed-43e2-4c2d-aafe-fba01ed5d04f__list_objects`
- `mcp__3e8ed9ed-43e2-4c2d-aafe-fba01ed5d04f__get_object`
- `mcp__3e8ed9ed-43e2-4c2d-aafe-fba01ed5d04f__put_object`

Bucket: `techlink-cowork-artifacts`

---

## Edge cases

- **No artifacts in S3**: Tell the user and offer to create the first team dashboard now.
- **`get_object` returns `{ found: false }`**: Skip and log. Do not error.
- **New artifact requires user approval**: Expected — Cowork shows an approval prompt. On first sync, user may see several.
- **HTML too large**: Warn if content exceeds ~500KB. Suggest simplifying the source HTML.
