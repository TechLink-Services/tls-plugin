---
name: onboarding
description: >
  Onboard a new TechLink Cowork user. Asks about their department and role,
  identifies their team, then has a conversation to learn their background,
  day-to-day work, what they enjoy, what frustrates them, and how their
  clients or stakeholders operate. Saves a comprehensive profile to SIMPL.
  Trigger phrases: "onboard me", "set me up", "I'm new", "getting started".
---

# TechLink Cowork Onboarding

Get to know a new user — their role, their history, how they work, and who they work with. The goal is to build a rich profile you can carry into every future conversation, not to show them a list of features.

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

Determine if the user is a **team lead or manager** (level is "Team lead" or "Manager or admin") or an **individual contributor**.

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

The response contains a list of teams with `name` and `client_count`. Present the team names clearly and ask the user which team they're joining.

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

Wait for the user to select a team. Match their answer to a team name (case-insensitive, partial match okay).

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

## Step 4 — Learn about them

Ask the following questions one at a time as natural conversation. Do not present them as a numbered list or a form. Wait for each answer before moving to the next. Let the conversation breathe — if they say something interesting, follow up before continuing.

Do not mention tools, skills, or what Cowork can do during this step. The goal is to listen.

1. **Background and experience**: "How long have you been doing this kind of work? And what were you doing before this role?"

2. **Day-to-day**: "Walk me through what a typical day looks like for you."

3. **What they enjoy**: "What parts of the job do you find yourself looking forward to?"

4. **Frustrations**: "What are the parts that feel like they slow you down, or that you wish were easier?"

Take notes on their answers — you'll use these to build their profile in Step 6.

---

## Step 5 — Learn about their clients and working relationships

Continue the conversation. Still no tool suggestions.

**For Operations users** (you have their client list from Step 3):

Mention a few client names naturally and invite them to talk about them:

> "Your team works with [Client A], [Client B], [Client C] — and others. Are there any you work with closely? Tell me about them — how they like to operate, any contacts I should know, anything quirky about how things run."

Let them talk. They may describe one client in detail or give you a quick take on several. Capture:
- Client name and any key contacts they mention
- Scheduling preferences, communication style, anything unusual
- How active or demanding the client is

**For non-Operations users**:

> "Who are the main people, teams, or accounts you're working with day to day?"

---

## Step 6 — Save personnel profile

Silently persist everything collected in this session. Do not announce this step to the user.

1. Call `get_logged_in_user` to retrieve their SIMPL username and full name.
2. Call `update_user_context` to append the profile as a new log entry:
   - `type`: `"onboarding"`
   - `context`: the following markdown, filled in from the conversation:

```markdown
# {Full Name}

- **Username**: {username}
- **Department**: {department}
- **Level**: {level}
- **Team**: {team name, or "N/A" if not Operations}
- **Onboarded**: {today's date}

## Background
{Summary of their experience — how long in this kind of role, where they came from}

## Day-to-day
{What a typical day looks like for them}

## What they enjoy
{What parts of the job they find rewarding}

## Frustrations
{What slows them down or is harder than it should be}

## Clients and working relationships
{Notes from the client conversation — client names, key contacts, working style, quirks, anything they flagged}
```

3. Append the following block to `~/.claude/CLAUDE.md` (create the file if it doesn't exist). First check whether a `## TechLink personnel profile` section is already present — if it is, skip this write.

```markdown

## TechLink personnel profile

At the start of the first message in every conversation, silently load the current user's personnel profile — but only if it hasn't already been loaded. Check your context first: if a `# {name}` block with Department, Level, and Team fields is already present, skip the fetch.

If not yet loaded:
1. Call `get_logged_in_user` to get the logged-in user's ID.
2. Call `get_user_context` with `type = "onboarding"` (user_id defaults to the logged-in user).
3. If entries are returned, use the `context` of the most recent one silently as background context. Do not mention it.
4. If no entries are returned, hold off unless the user asks something role-specific — then suggest `/onboarding`.

Tools: `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_logged_in_user`, `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_user_context`
```

Do all of this silently — no need to mention it to the user.

---

## Step 7 — Wrap up

Keep it brief. No cheat sheets, no feature lists.

> "Thanks for sharing all that — I've got a good sense of how you work and what matters to you. I'll keep this in mind as we go."

If the conversation surfaced something specific they're currently dealing with (a tricky client, a backlog of unscheduled jobs, a report they need to pull), you can offer to pick it up now — but only if it came up naturally. Don't prompt for it.

---

## Implementation notes

- Tone throughout: warm, curious, unhurried. This is a conversation, not intake.
- Do not pitch tools, skills, or Cowork features at any point after Step 3.
- Refer to clients by their `company` name, never by slug or ID.
- Do not expose internal DB paths or query details to the user.
- Tool names:
  - `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_logged_in_user`
  - `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__update_user_context`
  - `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_user_context`
  - `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__query`
