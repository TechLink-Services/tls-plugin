---
name: mesmerize-confirm
description: >
  Generate a Mesmerize site confirmation email for a given work order number.
  The skill pulls work order details, site name and address, scheduled install date/time,
  and equipment tracking info from SIMPL, then formats a ready-to-send confirmation email.
---

# Mesmerize Confirm

Produce a confirmation email for a Mesmerize work order. The email confirms the scheduled install time, verifies equipment delivery, and asks the site contact to confirm all details.

## Step 1 — Get the work order number

If the user didn't provide a WO number, ask for it now. Otherwise proceed immediately.

## Step 2 — Gather all data in parallel

Run all of the following at the same time (they are independent):

**A. Work order details** — call `get_workorder(id: <wo_number>)`

Key fields to extract:
- `id` — the WO number
- `site_id` — used to look up site address and timezone
- `install_start` — the scheduled date/time (format: "YYYY-MM-DD HH:MM:SS"); if it is "0000-00-00 00:00:00" the job is not scheduled yet
- `statcode` — current status

Match the statcode the statuses from the workstatus table using this query:
```sql
select id, name from workstatus;
```

**B. Site details** — call `get_site_by_workorder(workorder_id: <wo_number>)`

Key fields to extract:
- `company` — the site name
- `siteid` — the site ID code (e.g. M54585)
- `id` — the site's numeric ID (use this for `get_site_address` in Step 2C)

**C. Logged-in user** — call `get_logged_in_user()` to get the sender's name (`fname` + `lname`).

## Step 3 — Get site address, timezone, and tracking (in parallel)

Once you have `site_id` from Step 2A (or `id` from Step 2B), run these two calls at the same time:

**A. Site address + timezone** — call `get_site_address(id: <site_id>)`

Key fields to extract:
- `address1`, `address2`, `city`, `state` (numeric ID — convert to abbreviation), `zip` — for the full address
- `timezone` — IANA timezone string (e.g. `America/Los_Angeles`); use this to determine the correct timezone abbreviation for `install_start`

**B. Equipment tracking** — call `get_tracking_records_by_workorder(workorder_id: <wo_number>)`

This returns an array of tracking records, each refreshed from the carrier API if stale. From the most recent record extract:
- `status` — human-readable delivery status
- `carrier` — carrier name
- `estimated_delivery_date` — use this as `{delivery_date}` in the email
- `permalink` — tracking URL
- `ship_to_city`, `ship_to_state` — delivery destination

## Step 4 — Format the scheduled date/time

Convert `install_start` into a human-friendly string using the site's `timezone` from `get_site_address`:

- Determine the correct timezone abbreviation from the IANA timezone string and the install date (accounting for daylight saving time). Examples:
  - `America/Los_Angeles` in summer → PDT; in winter → PST
  - `America/New_York` in summer → EDT; in winter → EST
  - `America/Chicago` in summer → CDT; in winter → CST
  - `America/Denver` in summer → MDT; in winter → MST
- Format: `DayOfWeek Month DD, YYYY H:MM AM/PM {tz_abbr}`
- Example: `Fri May 15, 2026 1:00 PM PDT`
- If not yet scheduled (zero date), output a clear placeholder so the user knows to update SIMPL before sending.

## Step 5 — Format the site address

Build the address string from `get_site_address` results:
- If `address2` is present and non-empty: `{address1}, {address2}, {city}, {state} {zip}`
- If no `address2`: `{address1}, {city}, {state} {zip}`

Note: the `state` field from `get_site_address` is a numeric ID. Convert it to the standard two-letter abbreviation (e.g. 32 → NV). If unsure, cross-reference with the city name.

## Step 6 — Compose and output the email

Output the subject line and body cleanly so the user can copy and send it. Pick "morning", "afternoon", or "evening" for the greeting based on the current time of day (user's local time).

---

Subject: REQUESTING CONFIRMATION – WO# {wo_id} - Site ID: {siteid} - Site Name: {site_company}

Good [morning/afternoon/evening],

We are reaching out today on behalf of Mesmerize to confirm our tech to install the media player for your advertising display at your site.
We are confirming the time and date of {formatted_install_date}.
We also need to confirm that you have received the equipment for this install.
We show it delivered on {delivery_date}.
Your site address is: {full_address}
Please confirm these details at your earliest convenience by replying all to this email or by calling 828-232-8419.

Thank you.
{user_fname} {user_lname}

---

## Edge cases

- **No tracking record**: Omit the delivery line entirely; note to the user that no tracking record was found and they should verify equipment status before sending.
- **Not yet scheduled**: Replace the date line with a clear note that the install date has not been set in SIMPL — the user must update it before sending.
- **Address not found**: Flag clearly in the output — do not guess or leave blank.
- **State is a numeric ID**: Convert to the two-letter abbreviation using the city/state context; do not show the raw number in the email.
- **Timezone unknown**: If `timezone` is missing from the site address, note that the timezone could not be determined and ask the user to confirm before sending.
- **WO note in `requested_time`**: If the `requested_time` field on the work order contains instructions (e.g. "ask for Hazel or Yvonne"), surface this as a highlighted note to the user above the email — do not include it in the email body.