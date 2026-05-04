---
name: Project Current State
description: Summary of SFMC Help Desk portal features, integrations, and known state. Live-launched 2026-04-27. Refreshed 2026-05-03.
type: project
originSessionId: 2fff6f8f-36dc-468a-bb26-9a471759b3e4
---

# Snapshot — 2026-05-03

Live on prod code 2026-04-27, prod Clerk since 2026-04-28 ~23:15 CT. Production stable. Polish + collab cycle continues; 2026-05-02 / 2026-05-03 were quiet. See CHANGELOG.md for user-facing copy on each item.

**2026-05-01 shipped (parent submodule pointer `2cb0191`, app commits `f724d9d` + `beea3e8`):**

Reply composer:
- **Spacebar fix in long replies.** RichTextEditor's value-sync useEffect was calling `editor.commands.setContent` on every keystroke whenever DOMPurify's round-trip produced byte-different HTML (whitespace normalization, attribute ordering, trailing-`<br>` markers). Each setContent reset the cursor mid-keystroke and ate the character — manifested as "press space, nothing happens / press again, the space disappears" once content grew past one paragraph. Added a `lastEmittedRef` to skip the sync when the incoming `value` matches what we just emitted; only re-set content for genuinely external value changes (form reset on send, canned-response insert). `src/components/shared/rich-text-editor.tsx`.

Agent presence (PR #3 by Liene, with three blocker fixes I added on top before merge):
- **Per-ticket presence** — `useTicketPresence(ticketId, currentUser)` joins channel `ticket-presence:<id>`, broadcasts `{userId, name, avatarUrl, isTyping}`, returns the other viewers. `<PresenceIndicator viewers={...} />` shows an avatar stack + count next to the title with a Tooltip listing names; green pulse dot per avatar when that viewer is typing.
- **Global presence** — `useGlobalPresence(currentUser)` mounts in the tickets layout; joins a single `global-ticket-presence` channel and tracks the current `activeTicketId` (added to `ui-store`). Returns `Map<ticketId, PresenceUser[]>` consumed by `TicketTable` to show an eye+count icon next to ticket IDs in every grid.
- **Agents/admins only.** Both hooks early-return for employees; presence isn't tracked or shown to them. (Note: presence channels themselves aren't role-gated server-side; an authenticated employee could in principle subscribe via console and see names + ticketIds. Documented as a known low-severity leak.)
- **Blocker fixes I applied** (commit `ee4e7a0`, squashed into PR #3 → `beea3e8`):
  1. **Wired `setViewingTicket`** — Liene's hook exported it but no caller invoked it, so `presenceMap` was always empty and the table eye icons would never appear. Replaced with `activeTicketId` slice in `ui-store`; detail page sets/clears on mount/unmount; hook re-tracks on store changes.
  2. **Typing-indicator effect** — was clearing its own auto-stop timer on every render so isTyping got stuck `true` after the user stopped typing, AND was firing `track(true)` on every keystroke. Refactored to fire only on false→true / true→false transitions; cleanup runs only on unmount; send-success path resets the transition ref.
  3. **PresenceIndicator** — had a third sibling under `<Tooltip>` (Trigger / Content / stray "is typing…" span). Tooltip only accepts Trigger + Content. Dropped the extra span (the tooltip already lists typing state per viewer).
  4. Bonus: Cleanup in `useGlobalPresence` now removes the supabase client too, not just the channel.

Process / convention:
- **One branch per fix or feature**, separately reviewable PRs (saved as `feedback_branch_per_feature.md`). Liene confirmed the rule on 2026-05-01 after combining changes earlier.
- **Liene's `liene-pagination` branch (commit `c72fca9`, on her fork)** is stale — it's a 4-line `sanitize.ts` fix from 2026-04-27 that's already in main and superseded (main now has `SAFE_URI_REGEXP` + `afterSanitizeAttributes` hook her version lacked). Don't ship; should be deleted from her fork. Branch name is misleading (it's an img-tag fix, not pagination).
- **Liene's `liene/fix-submit-as-status` (commit `71300fa`)** referenced in her email is **not pushed** anywhere reachable — not on origin, not on her fork. Awaiting push before review.

Known CI state:
- GitHub Actions CI has been red on main for 3+ builds with **pre-existing lint debt** (chronic `react/no-unescaped-entities` in `ticket-sidebar-panel.tsx:513`, `Unexpected any` in older code, unused imports). Vercel doesn't gate on it, deploys go through. Cleanup queued — pending `/schedule` once Shawn re-auths claude.ai (it failed with "API accounts not supported" 2026-05-01).

**2026-04-30 shipped:**

Ticket detail UX:
- **"Take it" button** in the sidebar Assignee section when a ticket is unassigned — single-click self-assigns. Hides once anyone is assigned.
- **Status-only submit works on every ticket** — dropped the `pendingStatus !== currentStatus` gate that left the Submit button disabled on Pending / On Hold / Solved tickets. Now any selected status is submittable; picking the current status is a silent no-op.
- **Visible confirmation flash** on the composer button after a status-only submit — `justSavedStatus` state turns the button green with "✓ Marked as <Status>" for 1.8s before resetting. Toast still fires as a secondary signal.
- **Right-edge padding** on detail-view content. The agent-views layout uses `-m-8` to negate the portal's main padding for full-width tables; that side-effected the detail view too. Detail children wrapper now has `px-4 py-2 lg:px-6 lg:py-4` so the sidebar cards don't slam into the viewport.

My Tickets:
- **Tabs** for agents and admins: "My Tickets (Created)" + "Assigned to Me", each with count chips. Employees keep the single grid (they don't take or get assigned tickets). Closed the open question raised after Take It shipped — agents had nowhere to find tickets they'd just claimed.

Inline system events in the conversation thread:
- **Migration 011** added `is_system boolean DEFAULT false` to the `messages` table (applied to prod). Indexed on `(ticket_id, is_system, created_at)` for the email reply filter.
- `/api/tickets/[id]/notify` route writes a system message in addition to firing email. Event types: `status_changed`, `assignment_changed`, `priority_changed`, `category_changed`, `subcategory_changed`, `department_changed`, `team_changed`. For `team_changed` the route resolves team uuids to names server-side. Email side stays restricted to status + assignment.
- `MessageThread` renders system rows as muted divider lines ("Shawn Fultz changed status from Open to Solved · Apr 30, 2026, 12:40 PM") via a separate `renderSystemEvent` path — no avatar, no card.
- `notifyNewReply` filters `is_system` rows out of the conversation embedded in reply emails. SLA `getActiveMetric` filters them out of `agentReplies` and `endUserFollowUps` so a status change doesn't reset the SLA timer. `useTickets` list query selects `is_system` so the SLA filter sees it. "X replies" header counts only non-system rows.
- **Race fix**: client explicitly invalidates the detail query after the notify POST resolves. Without this, the tickets-UPDATE realtime broadcast invalidated + refetched ahead of the server's `messages` INSERT, leaving the system row out of the rendered thread until manual refresh. Realtime stays as a backstop. Pattern worth remembering: when a single user action triggers BOTH a `tickets` mutation AND a downstream `messages` insert via a separate API call, the realtime ordering can lose to the local refetch — invalidate explicitly after the dependent call.

Things that intentionally do NOT emit a system line yet (raised but deferred): unassigning a ticket, CC add/remove, collaborator add/remove, title/description edits, attachment add, merge/unmerge. Cheap to add when wanted.

UUID display fix:
- Base UI `<Select.Value>` foot-gun strikes again — Team picker in ticket sidebar and Assign-to-User / Assign-to-Team in Admin → Routing were rendering raw uuids in the trigger. Added `children` render fns mapping value → name on each. (`feedback_base_ui_select_value.md` already covered this; just three more spots to apply it.)

Date+time in grids:
- Ticket-list "Created" column, Recent Requests "Updated" column, attachment timestamps, merge-modal preview now use `formatDateTime` (e.g. "Apr 30, 2026, 3:42 PM"). Date-only displays kept where appropriate (whats-new headers, holiday list, monthly chart axis labels).

Memos to business contact (sent today): one for date-and-time fix, plus the email-notifications memo from yesterday is still in flight. Pattern noted: when sending a "what changed" note to the business user, match the casual tone of the previous message and explicitly call out what is intentionally NOT changing — it heads off "did you forget X" follow-ups.

**2026-04-29 shipped:**

SLA system overhaul:
- The "8h/24h on every ticket regardless of policy" bug was caused by ticket-list / view-filter / cron consumers calling `getSlaStatus(t)` without `policies` and `schedules`, falling through to a hardcoded priority table (`SLA_CONFIG`). Wired policies + schedules into every consumer, then deleted `SLA_CONFIG` entirely. When no policy matches now → returns `null` (no SLA badge). Admins must configure a catch-all `any/any/any` policy if they want a global default.
- `SlaPolicyMetrics.firstReplyHours` and `nextReplyHours` are now `number | null`. Each metric has a "Track" Switch in the admin editor — off = N/A. Calculator returns `null` for null-metric tickets.
- `is_default` no longer gates the delete button; any policy can be deleted (with confirm). "Built-in" badge stays as informational only.
- Migration 010 also dropped the dead `ticket_types_handled` column on profiles (yesterday's cleanup).
- `<SlaIndicator emptyState="verbose" | "compact" | "hidden">` for the no-match state. Sidebar uses verbose, table cells use compact dash with tooltip.

Role-scoped UI scoping (4 separate fixes):
- `tickets/layout.tsx` early-returns children only for employees (no Views sidebar / queue chrome). Bare `/tickets` redirects employees to `/my-tickets`.
- `tickets/layout.tsx` also early-returns children for `isCreateNew = pathname === '/tickets/new'` — applies to ALL roles (admins shouldn't see the queue while filling out a creation form).
- `<TicketTabs />` (the editor-style multi-tab strip showing recently opened tickets) was rendered in `(portal)/layout.tsx` so it appeared on every portal page. Moved into `tickets/layout.tsx` and gated to `pathname.startsWith("/tickets")` + `!isEmployee`. Now sits at the top of the right-of-Views-sidebar pane only.
- "What's New" is admin-only: sidebar link gated on `isAdmin`, plus `/whats-new(.*)` added to the admin-only matcher in `middleware.ts` so non-admins navigating direct get redirected.

Email notification gaps:
- New tickets routed to a team queue with no specific agent now fan out to every active team member (agents + admins, skipping OOO + the requester) using a new `ticketCreatedTeam` template. Was completely missing — agents got zero email for new tickets since the no-agent-auto-assign change.
- CC'd users now get a "You've been CC'd" email at creation time AND when added later. The runtime-add path used to be a direct `ticket_cc` insert from the client (no server hop = no email possible); added `POST /api/tickets/[id]/cc` route doing access check + idempotent insert + `notifyCcAdded`. Client `handleCcAdd` now calls it.
- `notifyStatusChanged` now fans out to all CC'd users in addition to the creator (skipping anyone who'd be self-notifying).

Memo to business contact (sent 2026-04-29) explaining the email enhancements in plain English — pattern was: "Agents now get email when ticket lands in team queue / CC'd users get email when added or on status change / what is intentionally NOT changing (internal notes stay agent-only, agent-to-agent reassignment doesn't email CC)."

**Earlier work (kept for reference):**

2026-04-28 evening shipped:

Cutover hotfixes:
- `src/hooks/use-profile-id.ts` (new) — client mirror of `getProfileId()`. Patched `useCurrentUser` + `useNotifications` to use `user.externalId ?? user.id` so client-side queries against `profiles.id` (legacy ids) succeed for migrated users.
- Server reads custom session-token claim `userId` (set via Clerk Sessions → Customize session token) for FK lookups; falls back to `auth().userId`.
- Supabase migration 007 applied (RLS helpers use `coalesce(profile_id, sub)`).

Composer fixes:
- Submit-as caret dropdown was a no-op — Base UI `Menu.Item` only accepts `onClick`, not Radix-style `onSelect`. Patched. Filed `feedback_base_ui_menu_item.md` (third Base UI footgun).
- Reply box click-anywhere focus + properly anchored placeholder via `@tiptap/extension-placeholder`. Removed brittle negative-margin div hack; `min-h` now lives on the contenteditable instead of the wrapper.
- In-flight progress sweep across the top of the composer (blue/amber).

Admin · Users:
- Removed `ticket_types_handled` from the User type, admin form, and `/api/tickets` profile mapping. Migration 010 dropped the column from `profiles` (defensive backfill into `departments` was a no-op). Routing model now: **team_ids** drives routing rules, **departments** drives the ticket-detail assignee picker filter. Closed Task #7.

Pre-cutover comms:
- `scripts/clerk-cutover-announce.mjs` (new) — sent the "tech change tonight" pre-announce email to 100 users via Resend.

**Clerk migration runbook**: complete details + the three cutover gotchas live in `project_clerk_migration.md`. Phase 11 stabilization items remain (drop `sub` fallback in RLS, rotate touched credentials, optionally theme prod hosted accounts portal, decommission dev Clerk after ~1 week).

**Critical Clerk JWT note**: Clerk treats `sub` as reserved in BOTH the session-token customizer AND named JWT templates. Cannot override directly; we surface legacy id via custom claims (`userId` for session, `profile_id` for supabase template). RLS helpers coalesce.

---

# Long-form state (refreshed 2026-04-22, still mostly accurate; cross-check before relying)

SFMC Help Desk Portal is a full Zendesk replacement for SFMC Home Lending (Plano, TX).

**Core Stack:** Next.js 15 + Supabase + Clerk + Vercel + Resend

**Key integrations:**
- Clerk: auth, user provisioning, session management, role sync via publicMetadata. **Migration in progress** to a separate prod Clerk instance — see `project_clerk_migration.md`. Currently uses dev (sk_test_*). Session token customized with custom claims `userId` + `metadata`. Supabase JWT template customized with `profile_id` claim. Per Clerk's reserved-claim restriction, we never override `sub` — RLS helpers use `coalesce(profile_id, sub)` to find the legacy id.
- Supabase: DB (20+ tables with RLS), Storage (private 'attachments' bucket with signed URLs, public 'branding' bucket), Realtime
- Resend: outbound email notifications (7 trigger types), inbound email replies via webhook
- Vercel: hosting, SLA cron (GET /api/sla/check)

**URLs:**
- Portal: https://support.sfmc.com (CNAME to Vercel)
- NEXT_PUBLIC_APP_URL in Vercel should be https://support.sfmc.com
- Resend webhook: https://support.sfmc.com/api/webhooks/inbound-email

**Email system:**
- Outbound from: support.sfmc.com (verified in Resend), EMAIL_FROM env var
- Inbound Reply-To: ticket+T-XXXX@reply.sfmc.com (separate subdomain to avoid CNAME/MX conflict)
- Resend webhook signing secret: RESEND_WEBHOOK_SECRET env var, verified via Svix. Set in Vercel prod.
- Inbound DNS/Resend state (as of 2026-04-22):
  - MX: reply.sfmc.com → inbound-smtp.us-east-1.amazonaws.com priority 10 (live)
  - reply.sfmc.com added and verified as a separate domain in Resend; receiving toggled on
  - Inbound route in Resend posts to https://support.sfmc.com/api/webhooks/inbound-email (NOT reply.sfmc.com — that host has no A/CNAME, only MX)
  - Outbound DKIM/SPF still live on support.sfmc.com only; outbound sends AS support.sfmc.com, inbound receives AT reply.sfmc.com
  - Pending client-admin: TXT/DKIM records on reply.sfmc.com (Resend generated them for domain verification; receiving works without them but they should be added for completeness)

**Client details:**
- Company: SFMC Home Lending, 5408 W Plano Pkwy, Plano, TX 75093
- Default timezone: America/Chicago (Central Time)
- Users have editable timezone in profile (display only, SLA uses department schedule timezone)

**Session policy (as of 2026-04-20):** 7-day max session + 3-day rolling inactivity timeout. Configured in Clerk Dashboard → Sessions (inactivity needs Pro plan). Documented in README.

**Business rule (as of 2026-04-20):** Tickets do NOT auto-assign to a specific agent on submission. `/api/tickets` always writes `assigned_to = null` and relies on team routing; agents claim from the queue. Do not revert this — it's explicit user feedback.

**Data model note:** Department/category taxonomy lives in the `department_categories` table and is surfaced via the `useDepartmentCategories` hook in `src/hooks/use-admin-config.ts`. Consumers (create ticket form, SLA admin page, routing, custom field conditions editor, **ticket sidebar Department/Category/Sub-category dropdowns as of 2026-04-21**) all read from that single source — NOT from the old static `DEPARTMENT_CATEGORIES` in `src/data/ticket-config.ts`.

**Deploy-detection banner (2026-04-21):** `GET /api/version` returns `VERCEL_GIT_COMMIT_SHA` (force-dynamic, no-cache). `src/components/layout/version-banner.tsx` polls it every 5 min and on window focus; when the response differs from the first-loaded value, it shows a sticky blue "new version available" banner with a Refresh button. Route is in the middleware's public-route list. Wired into `src/app/(portal)/layout.tsx` above `AssumeUserBanner`.

**Ticket sidebar auto-save (2026-04-21):** Every sidebar field edit in `src/app/(portal)/tickets/[id]/page.tsx` goes through `handleUpdateField`, which calls `updateTicket.mutate` immediately and fires a 1.5 s success toast on save (error toast on failure). When adding new editable sidebar fields, route them through the same handler and add a label to `FIELD_LABELS` so the toast reads correctly. `useUpdateTicket` accepts `status | priority | category | ticketType | subCategory | assignedTo | assignedTeam | internalNotes`.

**Attachment size cap (2026-04-21):** 20 MB. Enforced client-side (`FileUpload` default `maxSizeMB=20`) and server-side (`MAX_FILE_SIZE` constant in `src/app/api/upload/route.ts`). Matches the Supabase Storage bucket cap Shawn raised in the dashboard.

**Features built:**
- Ticket CRUD via POST /api/tickets (with team-only auto-routing, CC, custom fields, attachments, mailing address)
- Reply via POST /api/tickets/[id]/reply (with canned response actions, @mentions, notifications)
- Merge tickets, follow-up tickets
- Admin: views, SLA policies, schedules, canned responses, users (create+edit+Clerk sync), routing rules, categories, custom fields (with display conditions), branding (with logo upload), regions/branches CRUD, bulk user import (ticket/Zendesk import was removed 2026-04-19)
- Conditional custom fields (2026-04-20): each field can have display-condition rules against ticketType / category / subCategory / priority; evaluator at `src/lib/custom-fields/conditions.ts`; admin editor at `src/components/admin/custom-field-conditions-editor.tsx`; migration `005_custom_field_conditions.sql` applied to prod DB.
- In-app `/whats-new` page backed by `src/data/changelog.ts` with sidebar link for all roles.
- Drag-and-drop file attachment onto ticket conversation (ReplyComposer exposes an imperative `addFiles(files)` via forwardRef; ticket detail page wires the drag listeners).
- Welcome emails on user creation (both bulk import and Add User): sends a Clerk sign-in token link via Resend. `welcomeUser` template + `notifyUserWelcome` in `src/lib/email/`.
- Assume User feature for admins (cookie-based, amber banner)
- User profile page with name edit (syncs to Clerk), avatar upload, timezone selector
- Signed URLs for attachments (1-hour expiry, access-checked)
- Image paste in reply composer with thumbnail preview
- Dynamic sidebar branding from DB (logo, company name, subtitle)
- Full email notifications: all sends awaited (Vercel requirement), conversation thread included in reply emails
- Inbound email processing (live + end-to-end verified 2026-04-22): Resend inbound route → /api/webhooks/inbound-email. Webhook payload is metadata-only; handler fetches the body via `resend.emails.receiving.get(email_id)`. HTML part is parsed preferentially via `parseHtmlReply` (src/lib/email/parse-reply.ts) which strips Gmail/Outlook/Apple quote blocks and signature divs before converting to plain text. Falls back to text part when no HTML. Replies from registered profile emails post to the ticket, solved tickets auto-reopen, unknown senders dropped silently with 200. Attachments not yet handled — low priority.
- Realtime ticket/message updates (live + verified 2026-04-22): Supabase Realtime publishes `messages`, `tickets`, `notifications` to the `supabase_realtime` publication. `useRealtimeTickets` hook in `RealtimeProvider` (portal layout) subscribes and invalidates TanStack Query cache on INSERT/UPDATE/DELETE. Open ticket views now auto-refresh when new messages (including inbound email replies) arrive. No polling — pure WebSocket.
- Requester-on-behalf (2026-04-22): Agents and admins can create tickets on behalf of another user via a new Requester field on the Create Ticket form. `POST /api/tickets` validates caller role and requester existence before overriding `created_by`. Employees cannot override. The selected requester is excluded from the CC picker.
- OOO preserves assignments (2026-04-22): Toggling Out of Office no longer bulk-unassigns the agent's open tickets. `POST /api/users/ooo` just flips the flag. Routing engine already filters OOO agents from team-member selection for new tickets. Sidebar warning text updated. See `feedback_ooo_preserve_assignments.md` — this is deliberate per dgonzalez, don't revert.

**Post-cutover stabilization (Clerk migration officially CLOSED 2026-04-29):**
- ~~Migration 009: drop the `sub` fallback~~ — Shawn decided 2026-04-29 to **leave the fallback in place for a few weeks**. Coalesce is harmless and the safety margin is cheap. Revisit ~mid-May 2026.
- Rotate credentials touched in chat during cutover: `RESEND_API_KEY`, prod Clerk webhook signing secret, Supabase DB password.
- Don't decom dev Clerk yet — keep ~1 week as a rollback target (earliest ~2026-05-05).
- Optional: theme prod hosted accounts portal (Clerk Dashboard → Customization) to match dev. Cosmetic, most users never see this page.

**Open items for tomorrow (2026-04-23):**
- **SLA first-response → next-response bug (dgonzalez #2, parked):** dgonzalez to share specific ticket IDs where the SLA feels wrong. Need to determine whether the *deadline itself* keeps ticking past the first agent reply (logic bug in `src/lib/sla/calculator.ts:50-77`), or whether the timer is correct but the UI label still reads "First Response" when it should say "Next Response." Two different fixes. Don't start investigating until he provides example tickets.
- **Inbound email attachments:** end-to-end email reply works and is verified, but attachments on inbound replies are dropped (text comes through, files don't). Low priority per Shawn — tackle when/if someone hits a real need.
- Timezone-aware date display throughout the portal via useTimezone() hook
- Attachment downloads via signed URL links
- Reports with monthly agent/department breakdown table
- SLA business-hours calculations respect department schedule timezone

**Test data state:** Migration `004_test_data.sql` was rolled back on 2026-04-21 — test profiles (prefix `test-`) and the 30 tickets they seeded were deleted. The file is still in the repo; Shawn may delete/rename it so it can't be re-run against production.

**Why:** Replacing Zendesk — needs full feature parity for internal mortgage lending support operations.

**Layout / long-string containment (2026-04-21):** The portal `<main>` has `min-w-0 overflow-x-hidden` and the ticket detail conversation pane / header have `min-w-0` on their flex children. This is what makes `break-words` actually break a 300-char no-space string. If you add a new top-level pane inside the portal and see horizontal scroll from long content, check that its flex ancestors have `min-w-0`.

**Card component tweak (2026-04-21):** `src/components/ui/card.tsx` — `Card` gets `has-data-[slot=card-header]:pt-0` and `CardHeader` uses `[.border-b]:py-4` (not just `pb-4`). This keeps a `bg-muted/50` CardHeader flush with the top of the Card instead of leaking Card padding above it. Fix applies globally across Admin/Profile.

**How to apply:** When making changes, ensure email notifications fire on all ticket mutations (must be awaited — Vercel kills unawaited async), signed URLs are used for attachments (never public), Clerk metadata stays in sync with Supabase profiles, Select onValueChange callbacks always wrap with null guard, and user-facing fixes are reflected in BOTH `CHANGELOG.md` and `src/data/changelog.ts` (see `feedback_changelog_pattern.md`).
