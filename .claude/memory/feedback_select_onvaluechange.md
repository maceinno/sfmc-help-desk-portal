---
name: Select onValueChange null guard
description: Base UI Select onValueChange passes string | null — never pass setState directly, always wrap with null guard
type: feedback
originSessionId: 2fff6f8f-36dc-468a-bb26-9a471759b3e4
---
Never pass a React setState function directly to the project's `<Select onValueChange={...}>`. The Select wrapper in `src/components/ui/select.tsx` uses `@base-ui/react/select` under the hood, and its callback receives `string | null` — not `string`. Passing setState directly fails the Vercel TypeScript build.

**Why:** Caught in production build — `Type 'string | null' is not assignable to type 'SetStateAction<string>'`.

**How to apply:** Always wrap: `onValueChange={(val) => { if (val) setState(val) }}` instead of `onValueChange={setState}`. This applies to all Select components in this codebase.
