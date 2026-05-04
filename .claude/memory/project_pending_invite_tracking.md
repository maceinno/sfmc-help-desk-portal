---
name: Pending: Invite tracking + backfill 9 users
description: Deferred work to add welcome-email/sign-in tracking and resend-welcome flow; 9 imported users need backfill once go-live dust settles
type: project
originSessionId: ba8da8dc-0fad-43cc-93ab-59dcd9ee0327
---
Deferred feature: invite/acceptance tracking on users page + resend-welcome capability. User chose option 1 (custom-token flow + DB tracking, not Clerk Invitations API) on 2026-04-24. Postponed because they went live 2026-04-27 and have higher-priority fixes to handle first.

**Why:** Currently no way to see if imported users got their welcome email or signed in. ~100 users now exist in dev Clerk (cap was hit during launch-day import) and most don't have working sign-in tokens or won't after the dev → prod migration invalidates them. Going forward, admins want visibility into invite state from the users table.

**Partial progress:** The Clerk migration's Phase 10 will re-blast welcome emails to all migrated users via `scripts/clerk-resend-welcome.mjs`. That's the same one-shot bulk action the deferred plan called for in step 7 — closes the urgent backfill need. Still leaves steps 1-6 (DB columns + invite-state UI + resend-from-users-table button) pending, but the user-facing pain is gone.

**How to apply:** Bring this up proactively when the user signals post-launch fires are under control (or when they ask "what was that thing we deferred"). Don't pester. When ready, the agreed plan is:

1. Add `welcome_email_sent_at` and `first_signed_in_at` columns to `profiles`.
2. Stamp `welcome_email_sent_at` in `src/app/api/import/users/route.ts` and `src/app/api/users/create/route.ts` after welcome email send succeeds.
3. Stamp `first_signed_in_at` from a Clerk `session.created` webhook (or extend existing user.* handler) if null.
4. Add Status column to `src/app/(portal)/admin/users/page.tsx`: Active (green) / Invite sent Xd ago (amber, red if >7d) / No invite (gray).
5. New `POST /api/users/resend-welcome` endpoint that mints fresh 7-day Clerk sign-in token, sends welcome email, stamps `welcome_email_sent_at`.
6. Per-row + bulk "Resend welcome" action in the users table.
7. Backfill the 9: either UI bulk-select, or one-off script targeting `profiles WHERE welcome_email_sent_at IS NULL AND first_signed_in_at IS NULL`.

Open clarifying questions when picked back up:
- Exact import date (so we can scope the backfill by `created_at` window if needed).
- Confirm both bulk backfill + permanent per-row resend button (recommended both).
