---
name: Clerk dev → prod migration (CLOSED 2026-04-29)
description: Historical post-mortem of the dev → prod Clerk cutover (succeeded 2026-04-28). Captures the three cutover-night gotchas + the JWT/RLS architecture (external_id pattern, profile_id custom claim, coalesce(profile_id, sub) RLS helpers). Reference only — migration is closed, no active work.
type: project
originSessionId: 4414eac5-bec7-4a8d-9d1d-2489db87d7a3
---

# Clerk dev → prod migration (CLOSED)

## STATUS (closed 2026-04-29)

**Cutover succeeded 2026-04-28 ~23:15 CT. Migration project closed 2026-04-29 by Shawn — no rollback planned.** Production is stable on the prod Clerk instance with the legacy id model intact via `external_id` + JWT custom claims. All 100 migrated users sign in via email-OTP, view tickets, post replies, view attachments without issue.

**Decision (2026-04-29):** Keep the `sub` fallback in RLS helpers for **a few weeks** rather than tightening to `profile_id` only. The coalesce pattern is harmless once everyone has `profile_id`, and leaving it in place gives a longer window if any pre-migration session somehow cycled back in. Revisit ~mid-May 2026; until then there's no migration 009.

**Outstanding stabilization items** (low urgency, not blocking):
- Rotate credentials touched in chat during cutover: `RESEND_API_KEY`, prod Clerk webhook signing secret, Supabase DB password.
- Decommission dev Clerk instance (keep ~1 week minimum as rollback target — earliest decom ~2026-05-05).
- Optional: theme prod hosted accounts portal to match dev (cosmetic; most users never see it).

The rest of this file is historical reference — runbook + the three gotchas, kept around so the next cutover doesn't re-discover them.

### Cutover-night gotchas (the things that actually broke and how we fixed them)

These are the issues no amount of pre-cutover prep caught. Future-you, if doing this again: check these BEFORE flipping the env vars, not after.

1. **Clerk JWT Template signing key** — when prod Clerk was cloned from dev, the supabase JWT template's HS256 signing key did NOT carry over the Supabase project's Legacy JWT Secret. Tokens signed → Supabase rejected the signature → user requests treated as anonymous → RLS denied all rows → 406 Not Acceptable on every `.single()` call.
   - **Fix**: Supabase Dashboard → Project Settings → JWT Keys → **Legacy JWT Secret** tab → Copy the secret. Paste into prod Clerk → JWT Templates → supabase → Signing key (eye/edit icon). Save. New tokens mint within ~60s with correct signature.
   - **How to detect this**: 406s on EVERY authenticated query, not just specific ones. JWT decodes but signature verification fails server-side.

2. **Session-token customizer ≠ JWT Templates** — Clerk has TWO separate places to customize claims, and they don't share state:
   - **Sessions → Customize session token** → drives `auth()` / `sessionClaims` server-side in Next.js
   - **JWT Templates → supabase** → drives the token returned by `getToken({ template: 'supabase' })` for Supabase

   When the prod instance was cloned, the supabase JWT template carried over but the session-token customizer didn't. Result: `getProfileId()` had no `userId` claim → fell back to `auth().userId` = new prod id → access checks like `ticket.created_by === userId` failed → 403 Forbidden on `/api/attachments/signed-urls` (and would have hit any other access-checked API route).
   - **Fix**: prod Clerk → Sessions → Customize session token → Claims editor → set `{"userId": "{{user.external_id || user.id}}", "metadata": "{{user.public_metadata}}"}`. Save.
   - **How to detect**: 403s on API routes that check ownership. Decode the session token at jwt.io — if there's no `userId` claim, that's it.

3. **Client-side `useUser().id` returns new prod id, not legacy** — the server-side `getProfileId()` helper was in place pre-cutover, but the client-side mirror wasn't. `useUser().id` is the prod Clerk id (no auto-coalesce with externalId). `useCurrentUser()` and `useNotifications()` queried `.eq('id', user.id)` against `profiles` and `notifications.to_user_id` — both stored legacy ids → 0 rows → 406.
   - **Fix**: shipped `src/hooks/use-profile-id.ts` returning `user.externalId ?? user.id`. Patched `use-current-user.ts` and `use-notifications.ts`. Submodule commit `b2030f5`, parent `4f9d1b8`.
   - **How to detect**: 406 on a specific `?id=eq.<value>` request — decode the URL and compare to actual `profiles.id` shape. If the URL has the new prod id (e.g. `user_3Czc...`) but profiles holds legacy ids (e.g. `user_3Cowr...`), this is the bug.

### Post-cutover follow-ups (Phase 11 — do these in next 24-48h)

- ⏳ **Drop the `sub` fallback from RLS helpers**. Migration 007 was written backward-compatible with `coalesce(profile_id, sub)`. Now that prod is stable and every authenticated user has `profile_id`, write a migration 009 that simplifies to `auth.jwt() ->> 'profile_id'`. Defensive but not critical.
- ✅ **Rotate credentials touched in chat tonight**: `RESEND_API_KEY`, prod webhook signing secret. Supabase DB password (touched on 2026-04-28 daytime). Confirmed rotated by Shawn on 2026-05-03; live values now in 1Password only.
- ⏳ **Theme prod hosted accounts portal** to match dev (Clerk Dashboard → Customization). Cosmetic. Most users never see this page.
- ⏳ **Decommission dev Clerk instance** once prod has been stable for ~1 week. Don't rush this — having dev as a rollback target for a few days is cheap insurance.

## TL;DR — pick up cold in 60 seconds

**Goal**: migrate Clerk auth from the dev instance (capped at 100 MAU, hit) to a fresh prod instance, preserving every legacy user id so Supabase data does NOT need to be rewritten.

**Approach**: re-create users in prod with `external_id = <legacy_dev_clerk_id>` (Clerk's documented dev→prod migration pattern, via `clerk/migration-tool`). Surface the legacy id back to the application and to Supabase as **custom JWT claims**, because Clerk forbids overriding `sub`:
- Session token → custom claim `userId` → app reads via `sessionClaims.userId` (helper `getProfileId()`).
- Supabase JWT template → custom claim `profile_id` → RLS helpers read `coalesce(profile_id, sub)`.

**Status as of 2026-04-28 ~22:30 CT (evening, pre-cutover)**: PAUSED before Phase 8 cutover. **Phases 5, 6, 7 complete + prod Clerk email confirmed working.** 100 users exist in prod Clerk with `external_id` populated. Shawn is sending a pre-announce email to existing users tonight, then will execute Phase 8 (Vercel env-var swap) after-hours.

What's done:
- ✅ DNS — all 5 CNAMEs verified green in Clerk Domains (Frontend API, Account portal, clkmail, both DKIMs). SSL certs issued.
- ✅ Phase 5: `dev-users.json` refreshed via `scripts/clerk-export-dev-users.mjs` (100 users, 82 emp / 11 agent / 7 admin, all clean).
- ✅ Phase 6: today's automated Supabase daily backup is the pre-cutover snapshot (Shawn confirmed, no manual one needed).
- ✅ Phase 7: `clerk/migration-tool` ran clean — 100/100 successful, 0 failed, ~30 sec. Verified via Backend API: each prod user has `external_id` = original dev clerk id (sample confirmed: dev `user_3Cowr8vKpSNrH3vhpQUPfwmpIZD` → prod `user_3CzcdyirLVEnGEm6ogP26Fg8ly1`, jmoreland@sfmc.com).
- ✅ **Prod Clerk email infra confirmed live** (2026-04-28 evening). Shawn navigated to `accounts.support.sfmc.com/sign-in` (the prod hosted accounts portal), entered his email, and received the 6-digit verification code. The earlier "Email DNS configured but email configuration failed" warning has cleared after CNAME propagation. **This means Phase 10 (welcome reblast) is now optional** — see "Email-OTP discovery" below.
- ✅ **Email-OTP flow discovery**: Shawn confirmed users have always used Clerk's `email_code` strategy (passwordless), not passwords. Post-cutover UX is therefore trivial — users just sign in with their email and a 6-digit code as they always have. Sessions on the dev instance invalidate at cutover; on next visit users just sign in normally and the new prod-issued JWT carries `userId` → `profile_id` → coalesced into RLS via migration 007 → existing data loads unchanged. **No password reset flow exists, none needed.**

What's NOT yet done:
- ⏳ Pre-announce email — Shawn sending tonight, generic "tech change" framing (no mention of auth backend or 100-user limit). Tone: "we're making a tech change tonight, you may need to sign in again, your data is unaffected." Decided 2026-04-28 evening.
- ⏳ Phase 8 — Vercel env-var swap (Shawn, after hours tonight). All three at once: `CLERK_PUBLISHABLE_KEY` (prod `pk_live_*`), `CLERK_SECRET_KEY` (prod `sk_live_*`), `CLERK_WEBHOOK_SECRET` (per 1Password — original value redacted from memory after rotation 2026-05-03). Then redeploy. **Active sessions invalidate at this moment** — that's why we're waiting for after hours.
- ⏳ Phase 9 — smoke test (private window, fresh prod sign-in token, load ticket, send reply, attach).
- ⏪ ~~Phase 10 — welcome-email reblast~~ → **DEMOTED to optional.** Email-OTP works on prod (verified), so users can sign themselves in without a fresh token. Keep `scripts/clerk-resend-welcome.mjs` as a tool but skip the mass blast. Send only to users who actually report trouble after cutover.

### Cosmetic note (non-blocking) — accounts portal theme

Shawn flagged a visual delta between dev and prod hosted sign-in pages (`accounts.support.sfmc.com/sign-in`):
- **Dev** — light theme, dark Continue button, orange "Development mode" banner. Theme was customized in Dev Clerk Dashboard → Customization.
- **Prod** — Clerk's default dark theme + purple Continue button. No customization yet applied.

Most users won't see this page — the in-app `<SignIn>` component at `support.sfmc.com/sign-in` reads `appearance` from app code (`ClerkProvider`) and is identical regardless of backing instance. The hosted portal is mainly used for account-management deep-links. To make them match cosmetically, copy the dev customization into Prod Clerk Dashboard → Customization post-cutover. Filed as nice-to-have, not blocking.

Migration tool is already cloned at `/tmp/migration-tool` with deps installed; if `/tmp` is wiped between sessions, re-clone from `https://github.com/clerk/migration-tool` and `bun install` (~5 sec). The export at `sfmc-internal-help-desk/dev-users.json` was also copied to `/tmp/migration-tool/dev-users.json` because the tool rejected the absolute path. Reuse from there if the migration ever needs to be re-run (e.g. via `--resume-after`).

## Why this migration is happening

- Clerk caps every dev instance at 100 MAU, regardless of plan tier (we're on Pro). On launch day (2026-04-27) bulk import hit the cap mid-upload — the user-visible error was "Forbidden — Clerk create failed" on every CSV row past 100.
- Pro-plan benefits apply to **prod** instances. Need to migrate to scale past 100.
- "Going live" was on the dev instance because we hadn't activated prod. That's the bug we're fixing.

## Decisions and rationale

### 1. external_id pattern (instead of DB rewrite)

`profiles.id` is `text` and is the FK target for ~12 columns:
- `tickets.created_by`, `tickets.assigned_to`
- `messages.author_id`
- `attachments.uploaded_by`
- `notifications.to_user_id`
- `routing_rules.assign_to_user`
- `ticket_cc.user_id`, `ticket_collaborators.user_id`
- `views.created_by`, `canned_responses.created_by`
- (full list in `supabase/migrations/001_initial_schema.sql`)

Rewriting all of these in a single transaction is risky. Instead we re-create users in prod with `external_id = legacy_dev_id` and surface the legacy id via JWT customization. Net: zero FK rewrite.

### 2. The TWO reserved-claim discoveries (2026-04-28)

Originally I (Claude) thought Clerk allowed overriding `sub` somewhere. **Both editors reject it as reserved**:
- 01:18 CT — tried `{"sub": ...}` in **Sessions → Customize session token** → `"You can't use the reserved claim: sub"`.
- 01:29 CT — tried it in the named **JWT Templates → supabase** template → SAME error.

Final workaround uses **custom claims** (different name in each editor for clarity):
- **Session token**: custom claim `userId` = `{{user.external_id || user.id}}`. App reads `sessionClaims.userId`. Helper `getProfileId()` returns it with `auth().userId` as fallback for non-migrated users.
- **Supabase JWT template**: custom claim `profile_id` = `{{user.external_id || user.id}}`. Migration `007_profile_id_jwt_claim.sql` rewrites the 5 RLS helpers (`get_current_user_id`, `get_user_role`, `get_user_team_ids`, `get_user_branch_id`, `get_user_region_id`) to use `coalesce(auth.jwt()->>'profile_id', auth.jwt()->>'sub')`.

The coalesce makes migration 007 backward-compatible: applied while users are on dev (no `profile_id` claim yet), it falls through to `sub` and behaves identically. Verified live by Shawn loading tickets at 01:35 CT — works.

### 3. Subdomain naming for prod Clerk

Clerk auto-derives the FAPI subdomain from the application domain — you don't pick it. Dev's app domain is `support.sfmc.com` (locked — Clerk explicitly does not allow changing dev domains per [docs](https://clerk.com/docs/guides/development/deployment/changing-domains)). Prod was created with the same `support.sfmc.com` in **Secondary application** mode → derives FAPI `clerk.support.sfmc.com`.

Both instances therefore share the same nominal FAPI in Clerk's system. The customer DNS CNAME for `clerk.support.sfmc.com` decides which Clerk edge actually serves traffic. **Cutover = swap CNAME ownership + Vercel env vars.**

For future Clerk apps under sfmc.com: use a distinct app subdomain so each one gets its own auto-derived auth subdomain (e.g. `crm.sfmc.com` → `clerk.crm.sfmc.com`). No flat-namespace conflicts.

### 4. Tooling

- [`clerk/migration-tool`](https://github.com/clerk/migration-tool) — official, accepts JSON. Each row's `userId` becomes `external_id` on the destination user. Supports passwordless users. Auto rate-limits when given a `sk_live_*` key (~3,500 users / 35 sec).
- The new Clerk CLI (`clerk init / config / deploy`) does **not** include user-migration commands. Skipped.

## What is already done

### Code shipped (all backward-compatible)

Submodule HEAD: `fa4fba4` (parent `500202e`). Stack of prep commits:

| SHA | Description |
|---|---|
| `5571d56` | chore: prep for Clerk dev → prod migration (external_id pattern) — adds resolveClerkId helper, scripts, webhook external_id support |
| `764917b` | chore: gitignore Clerk migration artifacts (PII / DB dumps) |
| `411b431` | fix: route auth().userId reads through getProfileId() helper — refactors 14 API routes |
| `fa4fba4` | fix: RLS helpers read profile_id JWT claim, fall back to sub — adds migration 007 |

Files of note:
- `src/lib/clerk/resolve-id.ts`:
  - `getProfileId()` — returns `sessionClaims.userId ?? auth().userId`. Use for any DB / FK lookup.
  - `resolveClerkId(client, idOrExternalId)` — legacy id → real Clerk id. Use when calling Clerk Backend API with a value sourced from `profiles.id`.
- 14 API routes refactored:
  - **Profile-only** (use `getProfileId()` directly): `upload/branding/route.ts`, `users/ooo/route.ts`, `tickets/[id]/notify/route.ts`, `attachments/signed-urls/route.ts`, `users/profile/avatar/route.ts`, `upload/route.ts`, `tickets/route.ts`, `tickets/[id]/reply/route.ts`, `tickets/[id]/merge/route.ts`.
  - **Profile + Clerk Backend API**: `users/profile/route.ts` (uses both), `users/sync-clerk/route.ts`, `users/assume/route.ts`, `users/create/route.ts`, `import/users/route.ts`.
- `src/app/api/webhooks/clerk/route.ts` — persists `profiles.id = data.external_id ?? data.id`. Update path keys on the resolved profile id. `data.id` (real Clerk id) still used for the Clerk-API `syncPublicMetadata` call.
- Middleware unchanged — only tests `userId` truthiness + reads `sessionClaims.metadata`. No FK use.

### Supabase migration 007 — APPLIED to prod

`supabase/migrations/007_profile_id_jwt_claim.sql` rewrites the 5 RLS helpers to `coalesce(profile_id, sub)`.

Applied 2026-04-28 ~01:25 CT against prod project `oygmgegnqenkecfsvhwt` via Management API:
```bash
jq -Rs '{query: .}' supabase/migrations/007_profile_id_jwt_claim.sql > /tmp/migration-body.json
curl -X POST -H "Authorization: Bearer $(cat ~/.supabase/access-token)" \
  -H "Content-Type: application/json" \
  --data-binary @/tmp/migration-body.json \
  "https://api.supabase.com/v1/projects/oygmgegnqenkecfsvhwt/database/query"
# returned [] HTTP 201
```

Verified: queried `pg_proc` post-application, all 5 helpers now use the coalesce form. Verified live by Shawn loading tickets without errors.

### Migration scripts (`sfmc-internal-help-desk/scripts/`)

| Script | Purpose | Status |
|---|---|---|
| `clerk-export-dev-users.mjs` | Dump dev Clerk users → JSON in migration-tool format | Already run; output in `dev-users.json` |
| `db-snapshot.sh` | `pg_dump --format=custom` wrapper | Limited use — server is Postgres 17, local pg_dump is 15 (version mismatch). Use Supabase Dashboard backup instead. |
| `clerk-resend-welcome.mjs` | Mint fresh prod sign-in tokens + email via Resend | Ready for Phase 10. Refuses non-prod keys without `--dry-run`. |

### Dev export (already captured)

- 100 users (dev cap was hit).
- All 100 have `email` + `firstName` + `userId` (becomes external_id on destination).
- Roles: 82 employees / 11 agents / 7 admins.
- File: `sfmc-internal-help-desk/dev-users.json` (gitignored, repo-local).
- Schema matches `clerk/migration-tool` exactly.

### Prod Clerk instance — fully configured (Shawn)

| Setting | Value |
|---|---|
| Application domain | `support.sfmc.com` (Secondary application mode) |
| Session token claims | `{"userId": "{{user.external_id \|\| user.id}}", "metadata": "{{user.public_metadata}}"}` |
| Supabase JWT template claims | `{"aud": "authenticated", "role": "authenticated", "email": "{{user.primary_email_address}}", "app_metadata": {}, "user_metadata": {}, "profile_id": "{{user.external_id \|\| user.id}}"}` |
| Webhook URL | `https://support.sfmc.com/api/webhooks/clerk` |
| Webhook events | `user.created`, `user.updated`, `user.deleted` |
| Webhook signing secret | Captured by Shawn → 1Password (don't store in repo or memory) |
| Email DNS warning | "Email DNS is configured but email configuration failed, please reach out to support" — likely clears after CNAMEs propagate. Our flow doesn't depend on Clerk-issued mail (welcome links go through Resend), so probably non-blocking. Reassess after DNS lands. |

### Outstanding pre-cutover items (Shawn — quick)

- [ ] **Restrictions check**: User & Authentication → Restrictions → confirm Allowlist is OFF (or `*@sfmc.com` if on). Today's "Forbidden" import errors were the dev cap, not Restrictions, so likely already OFF.
- [ ] **Supabase backup**: Project Settings → Database → Backups → confirm today's daily exists (Pro plan). Optionally click "Create backup" for an explicit pre-cutover snapshot.
- [ ] **Webhook signing secret in 1Password** (already captured per Shawn — confirm).

## DNS records pending (IT ticket payload)

All five are CNAMEs. No TXTs. All on subdomains under `support.sfmc.com` that should currently be non-existent. Resend's existing SPF/DKIM on `support.sfmc.com` is unaffected (Clerk uses child subdomain `clkmail.*` for outbound; selectors differ).

| # | Host | Type | Value | TTL |
|---|---|---|---|---|
| 1 | `clerk.support.sfmc.com` | CNAME | `frontend-api.clerk.services` | 300 |
| 2 | `accounts.support.sfmc.com` | CNAME | `accounts.clerk.services` | 300 |
| 3 | `clkmail.support.sfmc.com` | CNAME | `mail.mqujng411o0z.clerk.services` | 300 |
| 4 | `clk._domainkey.support.sfmc.com` | CNAME | `dkim1.mqujng411o0z.clerk.services` | 300 |
| 5 | `clk2._domainkey.support.sfmc.com` | CNAME | `dkim2.mqujng411o0z.clerk.services` | 300 |

Notes for IT: do not modify any existing records on `support.sfmc.com`. Verify each subdomain doesn't currently resolve before adding. TTL 300 (5 min). Verification in Clerk dashboard usually completes within 10–15 min of records going live.

## Cutover runbook (Phases 5–11)

> **Live status (2026-04-28 ~12:00 CT)**: Phases 5, 6, 7 complete. Pick up at Phase 8 tonight after hours.

Run when all 5 CNAMEs verify green in Clerk → Domains. Whole thing ~30 min.

### Phase 5 — Refresh dev export (Claude) — ✅ DONE 2026-04-28 11:51 CT

Re-run to capture any users added since the original snapshot (likely none — dev capped):
```bash
cd /home/shawn/dev/sfmc-help-desk-portal/sfmc-internal-help-desk
CLERK_SECRET_KEY=sk_test_xxx node scripts/clerk-export-dev-users.mjs --out dev-users.json
```
Sanity check: count, sample shape, no missing emails. (Already verified at 100 users — re-verify post-refresh.)

### Phase 6 — Pre-cutover backup (Shawn) — ✅ DONE 2026-04-28 (today's daily backup is the snapshot)

Recommended: Supabase Dashboard → Project Settings → Database → Backups → "Create backup". 1 click.

Alternative (won't work today): `./scripts/db-snapshot.sh pre-clerk-migrate` — fails because local `pg_dump` is v15, server is v17. To make this work, install postgresql-client-17 first.

### Phase 7 — Migrate users via clerk/migration-tool (Claude) — ✅ DONE 2026-04-28 11:58 CT (100/100 successful)

```bash
cd /tmp
git clone https://github.com/clerk/migration-tool
cd migration-tool
bun install
bun migrate -y -t clerk \
  -f /home/shawn/dev/sfmc-help-desk-portal/sfmc-internal-help-desk/dev-users.json \
  --clerk-secret-key sk_live_xxx
```
~35 sec for 100 users. Errors log to `./logs/`. Auto-detects `sk_live_*` and applies prod rate limits (100 req/sec, ~9 concurrent).

Verify post-run: spot-check 2–3 users in prod Clerk dashboard. Each should have `external_id` set to the original dev `user_xxxxx` id, and `email` matching.

### Phase 8 — Cutover (Shawn) — ⏳ PAUSED, deferred to after-hours 2026-04-28 evening

Vercel project → Settings → Environment Variables → swap **all three** at once, then redeploy:
- `CLERK_PUBLISHABLE_KEY` → prod `pk_live_*`
- `CLERK_SECRET_KEY` → prod `sk_live_*`
- `CLERK_WEBHOOK_SECRET` → the prod webhook signing secret from 1Password

Trigger redeploy. **All active sessions invalidate at this moment** — different JWT signing keys. Schedule for low-activity if possible (post comms first).

### Phase 9 — Smoke test (Claude + Shawn)

Private window:
1. Open `support.sfmc.com` (no session).
2. Claude mints a fresh prod sign-in token for one test user (Shawn's account is fine):
   ```bash
   curl -X POST -H "Authorization: Bearer sk_live_xxx" \
     -H "Content-Type: application/json" \
     -d '{"user_id": "user_<prod_clerk_id>", "expires_in_seconds": 600}' \
     https://api.clerk.com/v1/sign_in_tokens
   ```
3. Sign in via `support.sfmc.com/sign-in?__clerk_ticket=<token>`. Verify:
   - Sign-in completes.
   - Profile loads (RLS read works → `profile_id` claim is being read).
   - A ticket loads in detail view.
   - Reply works (insert with `author_id = profile id`).
   - Attachment upload works (`uploaded_by = profile id`).
4. **If broken**: revert all 3 Vercel env vars to dev values, redeploy. Dev instance is untouched and resumes serving (existing sessions still invalidated though). Investigate likely culprits: JWT template typo, webhook secret mismatch, env var typo.

### Phase 10 — Re-blast welcome emails (Claude) — **OPTIONAL after email-OTP confirmation**

Originally planned because we assumed users needed a fresh sign-in token. **No longer required**: Clerk's email-OTP (`email_code`) strategy works against prod immediately — users just sign in with their email and code as always. Their dev session invalidates at cutover; next visit they sign in normally, RLS coalesces `profile_id` into the legacy id, all data loads.

Keep this for stragglers who report trouble (rare). Same script, same params:
```bash
cd /home/shawn/dev/sfmc-help-desk-portal/sfmc-internal-help-desk
CLERK_SECRET_KEY=sk_live_xxx \
RESEND_API_KEY=re_xxx \
EMAIL_FROM='SFMC Help Desk <support@support.sfmc.com>' \
APP_URL=https://support.sfmc.com \
node scripts/clerk-resend-welcome.mjs --filter-external-id-only --user-email=<one-user>
```
The deferred `project_pending_invite_tracking.md` backlog item still applies if a future bulk reblast is needed.

### Phase 11 — Stabilize (later, 24-48h after stable)

Follow-ups when prod has been stable for a day or two:
- New migration: drop the `sub` fallback from RLS helpers (now always have `profile_id`).
- Rotate the prod webhook signing secret (it touched chat tonight).
- Rotate the Supabase DB password (also touched chat).
- Decommission dev instance.

## What Claude needs from Shawn at each cutover phase

| Phase | Required from Shawn |
|---|---|
| 5 | dev `CLERK_SECRET_KEY` (`sk_test_*`) — pasted earlier in 04-28 chat. Working memory only. |
| 6 | Supabase Dashboard backup confirmed, OR `SUPABASE_DB_URL` (already used at 01:25 CT). |
| 7 | prod `CLERK_SECRET_KEY` (`sk_live_*`). |
| 8 | prod `CLERK_PUBLISHABLE_KEY` (`pk_live_*`), `CLERK_SECRET_KEY` (`sk_live_*`), `CLERK_WEBHOOK_SECRET` (prod's). |
| 9 | Smoke-test login + spot a ticket + reply + attachment. |
| 10 | prod `CLERK_SECRET_KEY`, `RESEND_API_KEY`, `EMAIL_FROM` (existing env value), `APP_URL` = `https://support.sfmc.com`. |

## Risks + mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Migration tool fails on some users | Low — data is clean (100/100 had email + firstName + id) | Tool logs per-row; supports `--resume-after`. Fix and re-run. |
| Active sessions invalidated at cutover | **Certain** | Schedule low-activity. Comms post if needed. |
| Vercel env vars half-swapped (publishable updated, secret not) | Medium — easy mistake | Update all 3 at once, save, redeploy once. |
| RLS broken because `profile_id` claim missing | Low | Coalesce fallback to `sub` is in place (migration 007 — verified live). Even if the supabase template was somehow rolled back, RLS works. |
| App reads `sessionClaims.userId` but it's null | Low | `getProfileId()` falls back to `auth().userId`. For non-migrated users, the two are equal. |
| DNS propagation delay > 30 min | Low | TTL 300. Verify in Clerk dashboard before phase 7. |
| Clerk's "Email config failed" warning persists post-DNS | Medium | Open Clerk Support chat. Our flow doesn't use Clerk-issued mail (Resend handles welcome), so non-blocking for cutover. |
| Migrated users see broken old welcome links | Mitigated | Phase 10 re-blasts fresh emails. |
| Webhook fires during cutover with stale signing secret | Low — short window | Webhook handler will 401, Clerk auto-retries. Just stay within the retry window. |

## Sensitive credentials touched in chat (rotate after stable)

NOT stored in repo, NOT stored in this memory file. Working memory + Shawn's 1Password only. After cutover stabilizes:
- Dev `CLERK_SECRET_KEY` (sk_test_*) — dies when dev is decommissioned, low urgency.
- Supabase DB password — rotate via Dashboard → Database → "Reset database password".
- Prod webhook signing secret — rotate via Clerk → Webhooks → "Roll signing secret", update Vercel env.
- `RESEND_API_KEY` — pasted in chat 2026-04-28 evening for the cutover announcement blast. Rotate via Resend Dashboard → API Keys, update Vercel + .env.local.

## Where to look when picking up cold

- **This file**: `~/.claude/projects/-home-shawn-dev-sfmc-help-desk-portal/memory/project_clerk_migration.md`
- **Latest submodule prep commit**: `fa4fba4` (parent `500202e`)
- **Scripts**: `sfmc-internal-help-desk/scripts/clerk-{export-dev-users,resend-welcome}.mjs`, `db-snapshot.sh`
- **Helper**: `sfmc-internal-help-desk/src/lib/clerk/resolve-id.ts`
- **RLS migration**: `sfmc-internal-help-desk/supabase/migrations/007_profile_id_jwt_claim.sql` (already applied to prod)
- **Dev export**: `sfmc-internal-help-desk/dev-users.json` (gitignored, 100 users)
- **Original chat that designed this**: 2026-04-27 evening through 2026-04-28 ~01:30 CT

## How to apply (for future Claude)

When Shawn says "DNS is in" or "let's do the Clerk migration":
1. Read this file end-to-end.
2. Confirm CNAMEs verify in Clerk → Domains (ask Shawn for a screenshot).
3. Run Phase 5 first (refresh dev export).
4. Coordinate Phase 6 with Shawn (Dashboard backup).
5. Run Phase 7 (migration tool).
6. Hand off to Shawn for Phase 8 (env-var swap).
7. Run Phase 9 with him (smoke test).
8. Run Phase 10 (welcome blast).
9. Update this file to mark phases done; flag the Phase 11 follow-ups.

If anything in the runbook contradicts current code or DB state, trust current state and update the runbook — the runbook is a snapshot in time.
