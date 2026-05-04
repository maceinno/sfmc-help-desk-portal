---
name: OOO preserves existing ticket assignments
description: Toggling Out of Office must not bulk-unassign the agent's existing open tickets — only prevent new assignments
type: feedback
originSessionId: 899f147f-1c0e-499e-b067-a545c1cc298b
---
Toggling Out of Office must NOT unassign the agent's existing open tickets. It just flips the `is_out_of_office` flag. The routing engine (`src/lib/routing/rule-engine.ts`) already filters OOO agents out of team-member selection for new tickets, which handles coverage.

**Why:** Feedback from dgonzalez (2026-04-22). Agents want to pick their own tickets back up when they return rather than hunt them down across the queue. The prior behavior (bulk unassign + internal message on every ticket) was too aggressive — it created extra admin work and broke continuity on in-progress issues.

**How to apply:** When editing `POST /api/users/ooo/route.ts` or the OOO toggle flow, resist the pull to "clean up" by reassigning tickets. The deliberate design is: OOO is a forward-only flag, not a bulk operation. If a future ask wants to *allow* an agent to voluntarily clear their queue when going OOO, make it an explicit separate action ("Return all my tickets to the queue") — not a side effect of the OOO toggle.
