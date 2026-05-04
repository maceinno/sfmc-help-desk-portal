---
name: Supabase Realtime publication requires explicit table enrollment
description: Supabase Realtime fires zero events until tables are added to the supabase_realtime publication — not on by default, not in migrations
type: project
originSessionId: 899f147f-1c0e-499e-b067-a545c1cc298b
---
Supabase Realtime uses a Postgres logical replication publication named `supabase_realtime`. Tables must be **explicitly added** to it before `postgres_changes` events fire. The publication exists from day one, but it starts empty — and there's no UI warning that your subscription is listening to a publication with no tables.

On SFMC Help Desk (project ref `oygmgegnqenkecfsvhwt`), the enabled tables are: `public.messages`, `public.tickets`, `public.notifications`. Enrollment was done 2026-04-22 via the Management API after discovering all three were missing (subscription was live in `src/hooks/use-realtime-tickets.ts` but no events ever arrived).

**Why:** Non-obvious because the subscription code looks correct, the Realtime connection reports `SUBSCRIBED`, and no errors surface anywhere. The symptom is "UI just doesn't update" — which looks like a frontend bug.

**How to apply:** When realtime isn't firing, FIRST check publication membership via `SELECT schemaname, tablename FROM pg_publication_tables WHERE pubname = 'supabase_realtime'`. If a table you're subscribing to isn't listed, add it with `ALTER PUBLICATION supabase_realtime ADD TABLE public.<table>`. No redeploy needed — clients pick up new events immediately. Also: if a new table is added to the schema that needs realtime, remember to add it to the publication — our migrations don't do this automatically.
