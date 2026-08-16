---
name: Validate a single email address
description: >-
  Check whether one email address is real before writing to it or storing it — submit it to
  uChecker, wait for the asynchronous check to finish, and read the good/bad outcome with
  its reason.
api: openapi/uchecker-email-api-openapi.yml
operations:
  - ValidationController_validateSingle
  - ValidationController_getTask
  - ValidationController_getTaskResults
  - ValidationController_getBalance
generated: '2026-08-16'
method: generated
source: >-
  openapi/uchecker-email-api-openapi.yml (operationIds verified verbatim),
  conventions/uchecker-conventions.yml, errors/uchecker-problem-types.yml
---

# Validate a single email address

Use this when an agent has captured one address — from a form, a signature block, a CRM
record — and needs to know whether the mailbox actually exists before sending to it.

**This API is asynchronous.** `POST /api/v1/validate/single` does not return a verdict; it
returns a `task_id`. The verdict arrives when the task completes.

## Before you start

- Base URL: `https://api.uchecker.net`
- Auth: `x-api-key: uk_…` on every request (or `Authorization: Bearer <JWT>`). Both grant
  full access. See `../authentication/uchecker-authentication.yml`.
- Each address checked costs **1 credit**, debited the moment it is queued — not when the
  result comes back.

## Steps

1. **Check you can pay for it.** Call `ValidationController_getBalance`
   (`GET /api/v1/account/balance`). If the balance is zero the submission in step 2 returns
   **403**, not 429 — uChecker meters by credit, never by request rate.

2. **Submit the address.** Call `ValidationController_validateSingle`
   (`POST /api/v1/validate/single`) with `{"email": "user@example.com"}`. Optional:
   `client_type: "api"`, and `webhook_url` if you would rather be called back than poll.

   > **Retry hazard.** This operation has **no** `idempotency_key`. If the request times out
   > and you retry, you create a second task and spend a second credit. When a retry is
   > possible, use `ValidationController_validateBulk` with a one-element `emails` array and
   > an `idempotency_key` instead — see the bulk skill.

3. **Poll the task.** Take `task_id` from the response and call
   `ValidationController_getTask` (`GET /api/v1/tasks/{taskId}`) every **5–10 seconds**, the
   interval the provider recommends. Watch the state:

   | state | code | meaning |
   |---|---|---|
   | `pending` | 0 | queued |
   | `processing` | 1 | running; `progress_percent` is live |
   | `completed` | 3 | done |
   | `failed` | -1 | stop and report `task_id` to support@uchecker.net |

   Do not poll faster hoping for a quicker answer. An SMTP check is a live conversation with
   the recipient's mail server; greylisting on large providers makes some answers slow by
   design.

4. **Read the result.** Call `ValidationController_getTaskResults`
   (`GET /api/v1/tasks/{taskId}/results`). Each item carries `validation_result`, which the
   contract defines as exactly two values:

   - `good` — the mailbox exists and accepts mail. Safe to send.
   - `bad` — it does not exist, is disabled, or the domain does not accept mail.

   For `bad`, the `result` field carries the reason: `mailbox_not_found`,
   `domain_not_found`, `smtp_rejected`, and others the provider does not enumerate. Do not
   hard-fail on an unrecognised reason string.

   > **Contract vs marketing.** uChecker's website describes a richer read — catch-all,
   > disposable, role-based, "unknown". **The API contract does not return those.** Build for
   > `good | bad`. See `../errors/uchecker-problem-types.yml`.

## Error handling

Errors are `{"success": false, "error": "<Russian text>"}` — no RFC 9457, no stable error
code. **Branch on the HTTP status, never on the message.**

| status | do this |
|---|---|
| 401 | Key or JWT bad. A JWT lives one hour; refresh via `AuthController_refresh`. |
| 403 | Out of credits. Top up; retrying will not help. |
| 404 | Task does not exist **or belongs to another account** — uChecker returns 404 for both. |
| 500 | Retry with backoff. Remember step 2 is not idempotent. |

## What this does not tell you

A `good` verdict says the mailbox exists. It says nothing about **consent**. A technically
clean address that never opted in is still spam, and providers detect that behaviourally.
Validation and permission are different problems.
