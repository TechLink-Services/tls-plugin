# TechLink Claude Plugin

## Personnel profile — load at conversation start

At the beginning of every conversation, silently load the current user's personnel profile from the simpl-db.

**Before querying, check your context.** If a `# {name}` block with Department, Level, and Team fields is already present in this conversation, skip the query — you already have it.

If not yet loaded:

1. Call `get_logged_in_user` (SIMPL connector) to retrieve their user ID.
2. Query the `user_context` table:
   ```sql
   SELECT context FROM user_context WHERE user_id = {user_id} AND type = 'onboarding'
   ```
3. If a row is found, use the `context` value silently as background context — you know their department, level, and team without them having to say so. Do not confirm or mention that you loaded it.
4. If no row is found, the user hasn't been onboarded. Hold off unless it becomes relevant, then suggest they run `/onboarding`.

### Tool names
- `mcp__36aa6fc7-8701-4c5a-9545-e3b8fc122992__get_logged_in_user`
- `mcp__81208da0-feb6-49af-ab5a-d1fdae279d7d__query`
