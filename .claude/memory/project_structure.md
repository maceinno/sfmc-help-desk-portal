---
name: SFMC Help Desk Portal Structure
description: Parent repo holds docs/context/CLAUDE.md files; two submodules for magic patterns and the main help desk app
type: project
originSessionId: b874f3b9-c521-4a3b-9da9-74df1fb920b9
---
sfmc-help-desk-portal is a parent monorepo intended to hold shared docs, context, and CLAUDE.md files.

It contains two git submodules:
- `sfmc-internal-help-desk-magic-patterns` → github.com/maceinno/SFMC-Internal-Help-Desk-Magic-Patterns
- `sfmc-internal-help-desk` → github.com/maceinno/sfmc-internal-help-desk

All repos live under the `maceinno` GitHub org.

**Why:** The parent repo centralizes documentation and AI context files while keeping the app code and design patterns in separate repos.

**How to apply:** When working on docs, CLAUDE.md, or shared context, operate in the parent repo root. When working on application code, operate inside the `sfmc-internal-help-desk` submodule.
