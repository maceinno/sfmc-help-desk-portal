---
name: Base UI Select.Value needs render children
description: Base UI's Select trigger shows the raw value unless Select.Value has a children render fn — silent footgun when values are UUIDs or other non-human strings
type: feedback
originSessionId: 68c7d022-67d0-43e8-b2ad-e18b72528108
---
This codebase's `<Select>` primitive is `@base-ui/react/select` (NOT Radix, despite the shadcn-style wrapper in `src/components/ui/select.tsx`). Base UI's `<Select.Value />` displays the **raw value** when no children are provided — it does NOT auto-resolve to the selected item's text the way Radix does.

**Why:** Shawn reported all dropdowns on /admin/users "show an id after selecting" on 2026-04-19. Root cause: `<SelectValue placeholder="Unassigned" />` was rendering the stored branch/region UUID after selection instead of the name. Fixed in commit 551f862.

**How to apply:** Whenever a Select's value is not already human-readable (UUIDs, numeric ids, lowercase status keys users shouldn't see), provide a children render function:

```tsx
<SelectValue placeholder="Unassigned">
  {(val: string) =>
    !val || val === '__none__' ? 'Unassigned' : getBranchName(val)
  }
</SelectValue>
```

Other places in the repo likely still rendering raw values: `src/app/(portal)/admin/routing/page.tsx` and `src/components/ticket-detail/ticket-sidebar-panel.tsx` (status/priority — cosmetically OK because the values already read well, e.g. "open"). Fix on sight if touched.

Base UI's own type definition confirms this: `SelectValueProps.children?: React.ReactNode | ((value: any) => React.ReactNode)` — the render-function form is the escape hatch.
