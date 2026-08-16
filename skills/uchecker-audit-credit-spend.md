---
name: Audit credit spend and job history
description: >-
  Answer "how many credits are left, what did we spend them on, and what did we buy" — the
  guardrail an agent needs before it is allowed to run validation jobs on someone's account.
api: openapi/uchecker-email-api-openapi.yml
operations:
  - ValidationController_getBalance
  - ValidationController_getStats
  - ValidationController_getTasks
  - ValidationController_getTaskAnalytics
  - BillingController_getPaymentHistory
generated: '2026-08-16'
method: generated
source: >-
  openapi/uchecker-email-api-openapi.yml and openapi/uchecker-default-api-openapi.yml
  (operationIds verified verbatim), conventions/uchecker-conventions.yml,
  plans/uchecker-plans-pricing.yml
---

# Audit credit spend and job history

uChecker gives an agent **no runtime budget signal**. There are no `RateLimit-*` headers, no
`X-Credits-Remaining`, nothing on the response to say how much room is left. The only way to
know is to ask — which makes this a mandatory pre-flight for any agent authorised to spend a
customer's credits, and the reconciliation loop afterwards.

## Steps

1. **Read the balance.** `ValidationController_getBalance` (`GET /api/v1/account/balance`).
   This is the whole budget. Compare it against `len(emails)` before submitting anything:
   1 address = 1 credit, debited **at enqueue**, and a bulk submission that exceeds the
   balance is rejected wholesale with **403**.

2. **Read account-wide usage.** `ValidationController_getStats`
   (`GET /api/v1/account/stats`) returns the summary plus a recent-jobs list
   (`AccountStatsLastList`). Use it to spot a runaway loop — repeated near-identical jobs are
   what a mis-keyed retry looks like from the outside.

3. **Walk the job history.** `ValidationController_getTasks` (`GET /api/v1/tasks`) is
   page-numbered: `page` (1-based, default 1) and `limit` (1–100, default 10). The response
   carries a `pagination` envelope with `page`, `limit`, `total` and `totalPages`. Loop until
   `page > totalPages`; there is no cursor.

4. **Attribute the spend.** For any job that looks large or unexpected, call
   `ValidationController_getTaskAnalytics` (`GET /api/v1/tasks/{taskId}/analytics`) to see
   what those credits bought — how many addresses came back deliverable and why the rest were
   rejected. A job with a very high rejection rate is a data-quality problem upstream, not a
   uChecker problem, and it is worth surfacing to the human.

5. **Reconcile against purchases.** `BillingController_getPaymentHistory`
   (`GET /api/v1/billing/history`, in the auth/billing split) lists credit purchases with the
   same `page`/`limit` pagination. Purchases in, credits consumed out.

## Spend guardrails for an agent

- **Never rotate the key.** `AuthController_resetApiKey` invalidates the current key
  instantly and would break every other integration on the account, including the MCP
  connection.
- **Never call the ESP surface** (`EspController_getPrice`, `EspController_provisionAccount`)
  from a customer workflow. It needs a separate ESP provider token, it creates and funds
  downstream accounts, and it carries no idempotency key.
- **Prefer bulk over single when retrying.** Only `ValidationController_validateBulk` accepts
  an `idempotency_key`; a retried single validation spends a second credit.
- Pricing is prepaid RUB packs from 0.20 ₽/address down to 0.05 ₽ at 500k+; the first 100
  addresses are free. See `../plans/uchecker-plans-pricing.yml`. There is no overdraft — when
  credits run out, work stops with a 403.
