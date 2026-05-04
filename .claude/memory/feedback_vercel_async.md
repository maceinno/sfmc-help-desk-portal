---
name: Vercel async requirement
description: Vercel serverless functions kill unawaited async work after response — always await email sends and other async operations
type: feedback
originSessionId: 2fff6f8f-36dc-468a-bb26-9a471759b3e4
---
Always `await` async operations (email sends, background DB writes) in API route handlers before returning the response. Vercel serverless functions terminate immediately after the response is sent — any fire-and-forget async work gets killed silently.

**Why:** Spent time debugging why emails weren't sending. The notifyNewReply() call was not awaited, so Vercel killed the function before Resend received the API call. No errors in logs because the function was terminated, not crashed.

**How to apply:** In any API route that triggers async side effects (email, webhooks, background DB operations), `await` the call before `return NextResponse.json(...)`. This applies to all `/api/` routes in this project.
