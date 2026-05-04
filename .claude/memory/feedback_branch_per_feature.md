---
name: One branch per feature/fix
description: Project convention — every fix or feature gets its own branch off main; never combine unrelated changes on the same branch
type: feedback
originSessionId: 72484ab1-58cf-4326-9509-c9e3fda330f6
---
One fix or feature per branch. Branches use the form `<contributor>/<feature-name>` (Liene: `liene/<feature-name>`; if I push directly, mirror the convention). Each PR must be reviewable and mergeable independently of any other in-flight work.

**Why:** Earlier in the project Liene combined unrelated changes on a single branch, which made review and rollback awkward. Shawn established the rule "Always create a separate branch from main for each fix or feature. Never combine unrelated changes on the same branch." Liene acknowledged the rule on 2026-05-01 and is now using it.

**How to apply:**
- When working on multiple unrelated changes, branch off main once per change and open one PR per branch.
- Don't piggyback an unrelated drive-by fix onto an in-flight feature branch — even small ones. Open a new branch.
- When fixing issues found in someone else's PR, push the fix as a commit *on their branch* (so the diff stays attributed) — that's still "one feature per branch" because the branch's purpose is that feature.
- The rule applies to me too. If I'm asked to do two unrelated things, create two branches, two PRs.
