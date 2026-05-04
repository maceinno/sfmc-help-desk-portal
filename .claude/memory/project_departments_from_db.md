---
name: Departments and categories come from the DB, not static config
description: Use useDepartmentCategories() — do not read from the legacy DEPARTMENT_CATEGORIES / TICKET_TYPES constants
type: project
originSessionId: fed867ec-4b98-40e1-805f-444fca4a90ff
---
The department/category/sub-category taxonomy lives in the `department_categories` Supabase table and is managed under **Admin → Departments & Categories**. Consume it via `useDepartmentCategories()` in `src/hooks/use-admin-config.ts`, which returns `{ ticket_type, categories: [{ name, subCategories? }] }[]`.

**Why:** Before 2026-04-20, the create-ticket form and SLA admin page read from static `DEPARTMENT_CATEGORIES` in `src/data/ticket-config.ts` and a separate hard-coded `TICKET_CATEGORIES` constant in the SLA page. Admin-managed categories added through the UI were ignored by both surfaces, which caused user reports that "Other" and new sub-categories weren't showing up on ticket creation. The fix unified everything on the DB hook.

**How to apply:** Any new surface that needs to pick a ticket type, category, or sub-category — SLA, routing, custom field conditions, views, reports filters, etc. — must source from `useDepartmentCategories()` (or a server-side equivalent select from `department_categories`). Do NOT import `DEPARTMENT_CATEGORIES` or the `TICKET_TYPES` array from `@/data/ticket-config` for dropdown options. That file still holds `DEFAULT_TICKET_TYPE_FIELD_CONFIGS` and `US_STATES` which are still used; the taxonomy constants themselves are legacy.
