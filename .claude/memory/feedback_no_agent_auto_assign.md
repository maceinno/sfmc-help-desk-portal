---
name: Tickets do not auto-assign to a specific agent
description: POST /api/tickets writes assigned_to=null; only team routing applies. Do not revert.
type: feedback
originSessionId: fed867ec-4b98-40e1-805f-444fca4a90ff
---
On ticket creation, `/api/tickets` must always write `assigned_to: null` even when a routing rule produces an `assignedTo`. Only the `assigned_team` side of the routing result is persisted. Agents claim tickets from the queue themselves.

**Why:** Shawn's users explicitly complained that auto-assigning to a specific agent on submission caused the wrong person to "own" the ticket before a human triaged it. Per Shawn's feedback on 2026-04-20: "Tickets should not auto assign to an agent when submitted. Should default to the department then can be taken by an agent." The `applyRoutingRules` function still runs (to resolve the team), but we discard its `assignedTo` output.

**How to apply:** If you touch the ticket-create API route, the routing engine, or anything around auto-assignment, preserve this behavior — never re-introduce agent-level auto-assign on creation. If an admin explicitly reassigns a ticket later via the sidebar UI, that's a separate code path and fine.
