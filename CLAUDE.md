# SFMC Help Desk Portal

## Structure
This is a parent repo with two git submodules:
- `sfmc-internal-help-desk-magic-patterns/` -- Design prototype (Magic Patterns)
- `sfmc-internal-help-desk/` -- Production application (Next.js 15)

## Purpose
The parent repo holds shared documentation and context. The actual application code lives in the `sfmc-internal-help-desk` submodule.

## Submodule URLs
- Magic Patterns: https://github.com/maceinno/SFMC-Internal-Help-Desk-Magic-Patterns.git
- Production App: https://github.com/maceinno/sfmc-internal-help-desk.git

## Quick Start
```bash
# Clone with submodules
git clone --recurse-submodules <this-repo-url>

# Or if already cloned without submodules
git submodule update --init --recursive

# Work in the production app
cd sfmc-internal-help-desk
npm install
cp .env.local.example .env.local  # fill in credentials
npm run dev
```
