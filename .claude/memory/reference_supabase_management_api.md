---
name: Supabase Management API access via cached CLI token
description: How to run arbitrary SQL against any of Shawn's Supabase projects without needing DB passwords or linking
type: reference
originSessionId: 899f147f-1c0e-499e-b067-a545c1cc298b
---
The Supabase CLI caches a personal access token at `~/.supabase/access-token` (format `sbp_...`) after `supabase login`. This token works against the Management API, which includes a SQL-execution endpoint.

**Run SQL against any project:**
```
TOKEN=$(cat ~/.supabase/access-token)
curl -sS -X POST "https://api.supabase.com/v1/projects/<project-ref>/database/query" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"SELECT ..."}'
```

**Find project ref:** `supabase projects list` lists all orgs/projects Shawn has access to. Or extract from any NEXT_PUBLIC_SUPABASE_URL in `.env.local` — the subdomain is the ref.

SFMC Help Desk project ref: `oygmgegnqenkecfsvhwt`.

**Why it's useful:** The service role JWT alone can't reach system catalogs (pg_publication_tables, pg_stat_activity, etc.) via PostgREST. The Management API SQL endpoint runs as superuser and can do anything. No need for DB password, `supabase link`, or a psql connection. Works for quick diagnostics (checking publications, RLS policies, indexes, row counts) and one-off schema changes.

**Caution:** This runs as superuser with full DDL/DML capability against production. Treat it the way you'd treat any prod DB access — verify queries are read-only before running, and prefer migration files for persistent schema changes so they stay in source control.
