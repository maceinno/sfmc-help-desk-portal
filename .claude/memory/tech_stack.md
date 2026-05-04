---
name: SFMC Help Desk Tech Stack
description: Confirmed technology choices for the production help desk app - Next.js, Supabase, Clerk, Vercel
type: project
originSessionId: b874f3b9-c521-4a3b-9da9-74df1fb920b9
---
Production app tech stack (confirmed 2026-04-13):
- **Framework**: Next.js 15 (App Router) with TypeScript
- **Database + Storage + Realtime**: Supabase (PostgreSQL + RLS + Storage)
- **Auth**: Clerk (email/password initially, SSO later). Clerk JWTs passed to Supabase for RLS.
- **Hosting**: Vercel (preview deploys on PRs, production on main)
- **Server State**: TanStack Query
- **Client State**: Zustand
- **Forms**: React Hook Form + Zod
- **UI**: shadcn/ui + Tailwind CSS v4 + Lucide icons
- **Charts**: Recharts
- **Testing**: Vitest + Testing Library + Playwright

**Why:** User confirmed Supabase, Clerk, and Vercel as the stack. Full foundation-first build approach.

**How to apply:** All new code should use these libraries. Don't introduce alternatives.
