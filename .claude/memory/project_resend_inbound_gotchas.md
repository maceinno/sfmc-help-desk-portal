---
name: Resend inbound email gotchas
description: Two non-obvious Resend quirks that blocked inbound email replies until we untangled them
type: project
originSessionId: 899f147f-1c0e-499e-b067-a545c1cc298b
---
**1. The email.received webhook payload is metadata-only.** It contains from/to/subject/email_id but NO body (no `text`, no `html`). Resend did this to keep webhooks under serverless payload limits. To get the actual body you must call `resend.emails.receiving.get(email_id)` (endpoint: `GET /emails/receiving/{id}`). Our handler at `src/app/api/webhooks/inbound-email/route.ts` does this fetch.

**2. The Resend API key needs "full access" to read received emails.** A send-only or narrowly-scoped key will fetch outbound fine but return an error on `emails.receiving.get`. Symptom: handler returns 500 with `"Failed to fetch email body"` even though the webhook is verified and the MX/DNS path works. Fix is in Resend dashboard → API Keys → use a full-access key for `RESEND_API_KEY` in Vercel env.

**Why:** Both are easy to miss because neither surfaces in the webhook event itself — the webhook will happily keep firing `email.received` events and retrying with 500s.

**How to apply:** When diagnosing inbound email failures, don't waste time looking for body content in the webhook payload. Go straight to the Vercel logs for the actual error from `resend.emails.receiving.get()`. If it's a permission error, the API key is the problem.
