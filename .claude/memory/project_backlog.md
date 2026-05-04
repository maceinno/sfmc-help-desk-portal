---
name: Project Backlog
description: Persistent post-launch backlog. Items deferred or patched-quickly that need a proper revisit. Append; don't overwrite.
type: project
originSessionId: 4414eac5-bec7-4a8d-9d1d-2489db87d7a3
---
Running backlog for SFMC Help Desk portal — survives session disconnects. Add new items at the top with date + context. Mark `[done]` and date when picked up.

---

## 2026-04-27 — Solving without a comment + "Thank you" reply treadmill on Solved tickets

**Status:** Shawn is thinking about it. Two-part proposal on the table; needs a decision before any code change.

**Pain points raised:**
1. Agents can't easily mark a ticket Solved without typing a reply. The composer's Submit-as button is disabled with an empty composer; the right-sidebar Status dropdown does work for status-only changes but isn't discoverable from the composer.
2. When an end user replies to a Solved ticket via inbound email — most often "Thank you!" — the inbound-email handler currently auto-reopens the ticket. Agents end up replying "you're welcome" just to re-solve it. This was an explicit annoyance in the old Zendesk flow.

**Proposed Option A — Inbound replies on Solved tickets stay Solved, but bubble.**
- Stop flipping Solved → Open in `src/app/api/webhooks/inbound-email/route.ts` (today it auto-reopens).
- Reply still posts to the thread; existing realtime + notifications still fire.
- Stamp a `has_post_solve_reply` flag (or a timestamp like `last_post_solve_reply_at`) on the ticket so the list view can show a dot/badge.
- New view "Replies on Solved" so an agent can scan and triage; the genuine-followup ones can be reopened from there.
- Tradeoff: a real "actually still broken" reply could sit unseen if the agent doesn't check the new view. Mitigate with: (a) 24-hour SLA on the new queue, or (b) email digest, or (c) make the badge prominent enough that agents see it on their normal queue.

**Proposed Option B — Empty-composer Submit button doubles as "Mark as <status>".**
- Today: button is `disabled={!hasContent || isSending}`.
- Change to: when composer is empty AND `pendingStatus !== currentStatus`, enable the button with label "Mark as <status>" — click sends a status-only update (no reply posted).
- When composer has content, behavior stays as today ("Submit as <status>" = reply + status).
- Tradeoff: changes a button that just shipped today; small risk of "wait, why is it enabled now?" surprise. Label change should make it obvious.

**My recommendation if/when picked up:** ship both. Option A solves the "Thank you" treadmill at the root; Option B makes the right-sidebar's existing functionality also work via the composer button without confusion. They're independent and don't depend on each other.

**Files to touch:**
- Option A: `src/app/api/webhooks/inbound-email/route.ts` (status-flip logic), DB migration for the new column, `src/components/tickets/ticket-list.tsx` + queue list (badge), new view config in `src/data/views.ts` (or wherever views are defined).
- Option B: `src/components/ticket-detail/reply-composer.tsx` (button disabled-state + label logic).

**Don't forget:** changelog (`CHANGELOG.md` + `src/data/changelog.ts`) for whichever ships, per the project pattern.

---

## 2026-04-27 — Tailwind Typography plugin (revisit rich-text rendering)
**Quick fix shipped today:** added `[&_ul]:list-disc [&_ul]:pl-6 [&_ol]:list-decimal [&_ol]:pl-6` to the existing `prose prose-sm` className in three places to make bullet/numbered lists render:
- `sfmc-internal-help-desk/src/components/shared/rich-text-editor.tsx` (live editor)
- `sfmc-internal-help-desk/src/components/ticket-detail/message-thread.tsx:124` (description render)
- `sfmc-internal-help-desk/src/components/ticket-detail/message-thread.tsx:257` (HTML reply render)

**Root cause:** code uses `prose prose-sm` but `@tailwindcss/typography` is NOT installed and not loaded via `@plugin` in `src/app/globals.css`, so the `prose` classes are no-ops. Tailwind v4 Preflight resets `<ul>`/`<ol>` padding+list-style, so markers render but get clipped.

**Why revisit:** the rich-text editor was added (commits `828b3be`, `9f9b431`) specifically to preserve formatting pasted from Word/Google Docs/Slack. Without the typography plugin, headings/blockquotes/paragraph spacing/code blocks pasted from those sources will look flat/wrong even though the HTML is preserved. Today's fix only handles lists.

**How to apply when picked up:**
1. `cd sfmc-internal-help-desk && npm i -D @tailwindcss/typography`
2. Add `@plugin "@tailwindcss/typography";` near the top of `src/app/globals.css` (Tailwind v4 plugin syntax).
3. Test pasting a formatted Word/Docs doc into both the new-ticket description and the reply composer; verify headings, blockquotes, code, paragraph spacing all look reasonable.
4. The inline `[&_p]:my-1 [&_ul]:my-1 [&_ol]:my-1 [&_ul]:list-disc …` overrides may now conflict with the plugin's defaults — decide whether to keep them (tighter spacing, our existing look) or drop them (let prose own it). Likely keep `[&_p]:my-1` for tighter messages, drop the list-style overrides since the plugin handles those.
5. Sanity-check `MessageThread` rendering and any other `prose` usage with `grep -rn '"prose' src/`.

---

## How this file works
- Append new backlog items at the top with a `## YYYY-MM-DD — short title` heading.
- Each item: what, why, how-to-apply, plus links to related files.
- This file is auto-loaded whenever Claude reads the memory index.
