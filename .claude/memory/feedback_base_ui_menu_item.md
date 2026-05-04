---
name: Base UI Menu.Item uses onClick, not onSelect
description: Base UI's @base-ui/react/menu Menu.Item has no onSelect prop — Radix-style onSelect handlers are silently dropped. Use onClick instead.
type: feedback
originSessionId: 4414eac5-bec7-4a8d-9d1d-2489db87d7a3
---
When wiring a click handler on a `DropdownMenuItem` in this project, use **`onClick`**, not `onSelect`. The dropdown is built on `@base-ui/react/menu` (not Radix). Base UI's `MenuItem` props include `onClick` but **no `onSelect`** — the `onSelect` prop is silently dropped by TypeScript's spread, the menu still closes on click, but your handler never fires. State doesn't update, button labels don't change, no error in the console.

**Why:** discovered during cutover-night triage 2026-04-28 when the reply composer's "Submit-as <status>" caret dropdown looked broken — items rendered, menu closed on click, but `setPendingStatus` was never called. shadcn-style `onSelect` is a Radix idiom that doesn't carry over to Base UI.

**How to apply:** anywhere you see `<DropdownMenuItem onSelect={...}>` in a `.tsx` file in this project, that's a bug — convert to `onClick`. Same goes for any other Base UI Menu primitives (`MenuPrimitive.Item`). Note this is **specific to Menu** — for Base UI's `Select`, `onValueChange` is correct (separate memory).

This is the third Base UI footgun in this codebase (after `Select.onValueChange` null-arg and `Select.Value` UUID render). When debugging "interactive control isn't firing," check the underlying Base UI prop signature against the type declarations under `node_modules/@base-ui/react/<component>/<part>/*.d.ts`.
