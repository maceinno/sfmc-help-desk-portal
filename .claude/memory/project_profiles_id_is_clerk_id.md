---
name: profiles.id is the Clerk user ID
description: profiles PK is a text column holding the Clerk user id — create the Clerk user first, then upsert with onConflict:'id' to avoid racing the user.created webhook
type: project
originSessionId: 68c7d022-67d0-43e8-b2ad-e18b72528108
---
The `profiles` table uses `id text PRIMARY KEY` populated with the Clerk user ID (not a UUID). Any server code that inserts into `profiles` must either:

1. Be triggered from the Clerk `user.created` webhook (already has the id), OR
2. Call `clerkClient.users.createUser({ skipPasswordRequirement: true, ... })` first and use the returned `clerkUser.id`.

The Clerk webhook in `src/app/api/webhooks/clerk/route.ts` also inserts a profile on `user.created`. So any direct insert from a server route **races with the webhook** — whichever loses hits a duplicate-key error.

**Why:**
- Bulk user import originally inserted without an id and hit `null value in column "id" of relation "profiles" violates not-null constraint` (2026-04-17).
- The prior `/api/users/create` treated duplicate-key as a fatal failure and **deleted the Clerk user** to "clean up" — silently destroying accounts when the webhook won the race. Fixed in commit 551f862.

**How to apply:**
- Never insert into `profiles` without an id.
- From server routes, always `supabase.from('profiles').upsert({ id, ... }, { onConflict: 'id' })` so webhook/route ordering doesn't matter.
- Orphan case (Clerk account exists, profile row missing): on `createUser` "already exists / taken", call `clerk.users.getUserList({ emailAddress: [email] })` and reuse `data[0].id` for the upsert. Don't delete the Clerk user in rollback if you reused a pre-existing id.
- For onboarding email after `createUser`: `skipPasswordRequirement: true` sends **no email on its own**. Generate a link via `clerk.signInTokens.createSignInToken({ userId, expiresInSeconds })` and send via Resend with `notifyUserWelcome` from `@/lib/email`. URL format: `${NEXT_PUBLIC_APP_URL}/sign-in?__clerk_ticket=${token.token}`.
