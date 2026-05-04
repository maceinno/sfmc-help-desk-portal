---
name: Verify before claiming UI/settings work
description: For UI fixes start the dev server and test in a browser; for settings recommendations confirm against the live target system. Don't ship-and-hope.
type: feedback
originSessionId: 4414eac5-bec7-4a8d-9d1d-2489db87d7a3
---
**Rule:** Don't claim a UI fix is done without testing it in a browser. Don't recommend a Clerk/Supabase/Vercel/DNS dashboard setting without verifying that the live editor actually accepts the value you're proposing.

**Why:** During the 2026-04-27/28 launch night, I twice shipped or recommended changes without verifying — both times Shawn caught it:
1. Shipped a "new-ticket form scrolls so Submit is reachable" fix on reasoning alone, no browser test. Shawn pushed back ("are you sure this is fixed? and why was the fix… actually fix it?"). The reasoning happened to be right, but the workflow was wrong.
2. Recommended `{"sub": "{{user.external_id || user.id}}"}` for the Clerk session-token customizer based on the migration docs — the dashboard editor rejects `sub` as reserved. Same with the named JWT template editor. Burned ~20 min of back-and-forth and required new code (custom `userId`/`profile_id` claims + DB migration 007) when a quick smoke test of the live editor would have surfaced the constraint upfront.

**How to apply:**
- **UI changes**: `npm run dev` in `sfmc-internal-help-desk`, navigate to the route, repro the issue, confirm visually, *then* commit + push. If I genuinely cannot test (e.g. no browser surface, requires real DB state), say so explicitly in the response — don't claim success.
- **Dashboard/settings recommendations** (Clerk, Supabase, Vercel, DNS provider): give Shawn the proposed value and ask him to try it before pasting it as the runbook step. If a recipe comes from external docs, flag the confidence ("this is what the docs say but I haven't verified against the live editor — try it first") rather than treating doc examples as ground truth.
- **Production migrations applied via Management API**: pair with a verification query immediately after, like checking `pg_proc` after a function rewrite or `information_schema.columns` after a default change. Don't skip the verification step.

The general principle: trust live state over docs/training data when they conflict, and assume they may conflict.
