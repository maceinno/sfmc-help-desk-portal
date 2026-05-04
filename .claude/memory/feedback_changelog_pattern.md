---
name: Changelog pattern for user-facing fixes
description: Every user-facing fix must be reflected in both CHANGELOG.md and src/data/changelog.ts so the in-app /whats-new page stays current
type: feedback
originSessionId: fed867ec-4b98-40e1-805f-444fca4a90ff
---
When shipping a user-visible fix or feature, add a plain-English entry to BOTH files before committing:
1. `sfmc-internal-help-desk/CHANGELOG.md` — docs mirror, newest on top, grouped by category
2. `sfmc-internal-help-desk/src/data/changelog.ts` — structured data read by the `/whats-new` page (sidebar link for all roles)

**Why:** Shawn asked on 2026-04-20 for a changelog "so the users know what was fixed", referencing a Zendesk-style screenshot. We built CHANGELOG.md as the source-of-truth doc and wired the same entries into an in-app page at `/whats-new` backed by `src/data/changelog.ts`. Both files must stay in sync — the docs file for repo history, the TS file for the rendered user-facing cards.

**How to apply:** On any fix that affects employee/agent/admin experience, append a new dated section (or add to the latest date's appropriate heading) in both files as part of the same commit. Keep language plain-English — employees should be able to read it without technical context. Skip the changelog only for pure internal refactors with no user-visible change.
