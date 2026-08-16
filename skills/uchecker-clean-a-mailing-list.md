---
name: Clean a mailing list in bulk
description: >-
  Submit a whole list to uChecker with an idempotency key, track the job to completion, read
  the deliverability breakdown, and download the cleaned good/bad lists for import into an
  ESP.
api: openapi/uchecker-email-api-openapi.yml
operations:
  - ValidationController_getBalance
  - ValidationController_validateBulk
  - ValidationController_getTask
  - ValidationController_getTaskAnalytics
  - ValidationController_getTaskResults
  - ValidationController_downloadCsv
  - ValidationController_downloadTaskResults
generated: '2026-08-16'
method: generated
source: >-
  openapi/uchecker-email-api-openapi.yml (operationIds verified verbatim),
  conventions/uchecker-conventions.yml, errors/uchecker-problem-types.yml,
  asyncapi/uchecker-webhooks.yml
---

# Clean a mailing list in bulk

The marquee flow. One submission, one task, one archive of cleaned lists. This is the
operation to reach for whenever more than a couple of addresses are involved — and also
whenever a **retry is possible**, because it is the only validation endpoint uChecker
protects with an idempotency key.

## Before you start

- Base URL: `https://api.uchecker.net`, auth header `x-api-key: uk_…`.
- 1 address = 1 credit, debited at enqueue time. Syntactically invalid addresses are
  **excluded free of charge** and returned in `invalid_details` — you are not billed for
  junk input, so there is no need to pre-clean the list.
- No request-rate limit exists. Your ceiling is the credit balance.

## Steps

1. **Size the job against the balance.** Call `ValidationController_getBalance`
   (`GET /api/v1/account/balance`). If `len(emails)` exceeds the credits available,
   `ValidationController_validateBulk` returns **403** for the whole submission — it does not
   partially process.

2. **Mint an idempotency key.** A UUID, or a deterministic job label like
   `import-2026-08-16-batch-3` (the provider's own example). Store it alongside your job
   record *before* you send.

3. **Submit the list.** Call `ValidationController_validateBulk`
   (`POST /api/v1/validate/bulk`):

   ```json
   {
     "emails": ["user1@example.com", "user2@example.com"],
     "client_type": "api",
     "idempotency_key": "import-2026-08-16-batch-3",
     "webhook_url": "https://your-site.com/webhook/validation-complete"
   }
   ```

   With the key set, a repeated submission returns the **current status of the existing
   task** instead of creating a duplicate and re-charging credits. That makes this call safe
   to retry on a timeout — the single-address endpoint is not.

   Read `invalid_details` on the response: those addresses never entered the job.

   > Volume: the REST endpoint declares no `maxItems` and uChecker's site quotes jobs up to
   > 9 million addresses. The **MCP** `validate_emails` tool caps a call at 10,000, so chunk
   > when going through MCP.

4. **Wait for completion.** Either:

   - **Poll** `ValidationController_getTask` (`GET /api/v1/tasks/{taskId}`) every 5–10 s and
     read `progress_percent`; or
   - **Receive the callback** on the `webhook_url` you supplied. Note that uChecker publishes
     **no payload schema and no signature** for that callback
     (`../asyncapi/uchecker-webhooks.yml`). Treat it as a *nudge*, not as data: when it
     arrives, go read the task. Whether a `failed` task fires the callback at all is
     undocumented.

5. **Read the breakdown before you download.** Call
   `ValidationController_getTaskAnalytics` (`GET /api/v1/tasks/{taskId}/analytics`) for the
   aggregate deliverability picture and the rejection reasons — SMTP rejection, no MX
   records, mailbox full. This is the number to report to a human before anything is
   deleted from their list.

6. **Take the results in the shape you need.**

   | need | operation | returns |
   |---|---|---|
   | structured, paged, in-process | `ValidationController_getTaskResults` | JSON (or CSV text with `format=csv`) |
   | a CSV file | `ValidationController_downloadCsv` | CSV download |
   | ready-to-import lists | `ValidationController_downloadTaskResults` | ZIP containing `good.txt` and `bad.txt` |

## After the check — what to actually do with the output

- `bad` addresses go on a **suppression list shared across every sending channel**, not just
  the marketing platform. The classic failure is suppressing an address for campaigns while
  transactional mail keeps hitting it from a different server.
- Re-import hygiene matters more than the check: dead addresses come back because the list is
  cleaned in one place and exported from another.
- Around 20% of a list decays per year. Validate at signup, before any large campaign after a
  month of idleness, and sweep the whole database twice a year.

## Error handling

Errors are `{"success": false, "error": "<Russian>"}`. Branch on status:
**401** auth, **403** insufficient credits, **404** unknown or foreign task, **500** retry —
and because step 3 carries the idempotency key, a retry after 500 is safe as long as you
resend the **same** key.
