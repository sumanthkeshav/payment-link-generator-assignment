# Payment Link Generation Workflow

## Overview

A modular n8n workflow that receives a payment request via webhook, generates a payment link through an external Payment Link API, and updates the payment record through an external Update API — with conditional retry handling, deduplication, and a full audit trail.

## User Story

As a dealership admin, I want to generate a shareable payment link for a customer directly from our system, so that I can collect payment without a manual, ad-hoc workaround — and know for certain whether the link was created and the payment record updated, even if the payment gateway or a downstream service is temporarily unavailable.

## Assumptions & Design Decisions

Every place the spec was ambiguous or the mock APIs didn't behave as documented, and the call made:

- **Modular sub-workflows**: `Generate Payment Link` and `Update Payment Record` each own their own conditional-retry policy and are independently reusable/testable, called from the orchestrator.
- **Retry classification**: transient failures (server errors, timeouts, connection drops) retry up to 3 times; requests the gateway definitively rejects fail immediately with no retry. Mirrors Stripe's/Razorpay's documented retry guidance — retrying a bad request just wastes time and delays the response.
- **Dedup key**: `referenceId`, checked against the audit Sheet before any external call — mirrors the idempotency-key convention documented by Stripe/Razorpay. `invoiceNumber` has no bearing on dedup; a caller wanting a new link for the same invoice must mint a new `referenceId`.
- **Audit trail**: every outcome (success, both failure types, duplicate, validation failure) is a row in a dedicated Google Sheet, keyed by `referenceId`, with per-phase status columns rather than one field overwritten at each step.
- **Alerting**: Slack fires for validation failures, unrecoverable link-generation/update failures, audit-log write failures, and the workflow-wide unexpected-crash catch-all — never for routine duplicate/replay hits, since those aren't operational problems.
- **Auth**: the webhook requires a shared-secret header, rejected before any workflow logic runs.
- **The mock APIs' live behavior doesn't fully match their documented shape.** The Payment Link API's real response omits a couple of documented fields. Both sub-workflows tolerate either shape and fall back to the caller's own `referenceId` when the API doesn't echo one. The Update API, by contrast, does match its documented shape.
- **The Update API's documented GET-with-body contract works against the real mock**, but this is a fragile pattern in general — some real infrastructure (observed: a load balancer fronting a public test API) rejects a body on a GET request before the application ever sees it.
- **`referenceId` is a local dedup key, not a forwarded upstream idempotency token.** It's checked against the audit Sheet before the first call, but never sent to the Payment Link API as an idempotency key.
- **The Slack alert URL is supplied via an environment variable rather than a stored credential**, since no credential type in this n8n instance can hold a bare URL usable in a node's own field.

## Architecture

Three n8n workflows: two independent, reusable sub-workflows and one orchestrator that ties them together.

```
Source system
   │ POST (with shared-secret header)
   ▼
┌─────────────────────────── Main Orchestrator ───────────────────────────┐
│                                                                          │
│  1. Webhook intake (header-auth rejects unauthenticated requests)       │
│  2. Validate required fields → 400 + audit log if missing               │
│  3. Dedup lookup by referenceId against the audit Sheet:                │
│       - no match          → continue                                    │
│       - already confirmed → short-circuit, return existing link (200)   │
│       - link generated but update unconfirmed → retry the update only  │
│  4. Log "received" row → call Generate Payment Link sub-workflow        │
│       - success → log "link_generated" row, continue                    │
│       - failure/crash → log "link_generation_unrecoverable",             │
│                         Slack alert, respond failure (no link)          │
│  5. Call Update Payment Record sub-workflow (using the new link)        │
│       - success → log "update_confirmed" row                            │
│       - failure/crash → log "update_unrecoverable" (link still shown),  │
│                         Slack alert, respond failure (with link)        │
│  6. Respond 200 with payment_link                                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
        │ Execute Workflow                    │ Execute Workflow
        ▼                                     ▼
┌─────────────────────────┐         ┌──────────────────────────┐
│ Generate Payment Link    │         │ Update Payment Record     │
│ (sub-workflow)           │         │ (sub-workflow)            │
│                          │         │                           │
│ POST → Payment Link API  │         │ GET-with-body → Update API│
│ Retry: 5xx/timeout/      │         │ Retry: same policy        │
│  connection → retry      │         │ Same shape-check pattern  │
│ 4xx/malformed → fail     │         │                           │
│  immediately, no retry   │         │                           │
│ 3 attempts, 2s wait      │         │ 3 attempts, 2s wait        │
│ Returns {success, data}  │         │ Returns {success, data}   │
│  or {success:false,      │         │  or {success:false,       │
│  error:{retryable,       │         │  error:{retryable,        │
│  reason, message}}       │         │  reason, message}}        │
└─────────────────────────┘         └──────────────────────────┘
```

### Main Orchestrator, section by section

#### Validation & Dedup

![Validation & Dedup](images/orchestrator-validation-dedup.png)

| Outcome | Trigger | What happens | Alert |
|---|---|---|---|
| Validation passed | All required fields present & well-formed | continues to link generation | none |
| Validation failed | Missing/malformed field(s) | rejected immediately, no processing | Slack |
| Duplicate — already completed | Same request already fully processed | existing link returned, nothing re-run | none |
| Duplicate — partially completed | Link was generated but the record update never confirmed | skips link generation, retries only the record update | depends on outcome |
| Not a duplicate | Nothing meaningful completed on a prior attempt | fully reprocessed from scratch | n/a |
| Audit log unavailable | The audit Sheet can't be read or written | request fails, nothing is silently lost | Slack |
| Unexpected crash | Any unanticipated internal error | request fails safely instead of exposing a raw error | Slack |

#### Payment Link Generation

![Payment Link Generation](images/orchestrator-link-generation.png)

| Outcome | Meaning | What happens |
|---|---|---|
| Link generated | Payment Link API succeeded | continues to record update |
| Rejected | Gateway declined the request outright — retrying wouldn't help | fails immediately, no retry |
| Malformed response | Gateway returned success but an unusable response | fails immediately, no retry |
| Temporarily unreachable | Connection issue, still unresolved after retries | fails after 3 attempts |
| Gateway unavailable | Gateway itself erroring, still unresolved after retries | fails after 3 attempts |
| Sub-workflow crash | Internal error instead of a clean result | fails safely, flagged distinctly from a gateway failure |

#### Update Payment Record

![Update Payment Record](images/orchestrator-update-record.png)

Same failure modes as link generation, with one difference: a failure here still surfaces the already-generated `payment_link` in the response, since that part already succeeded.

| Outcome | What happens |
|---|---|
| Update confirmed | Success — response includes the payment link |
| Update unrecoverable | Failure — response still includes the payment link, so nothing already accomplished is lost from view |

### Sub-workflows

Both sub-workflows share the same conditional-retry state machine; only the target API differs.

#### Generate Payment Link

![Generate Payment Link](images/subworkflow-generate-payment-link.png)

#### Update Payment Record

![Update Payment Record sub-workflow](images/subworkflow-update-payment-record.png)

| State | Condition | Next action |
|---|---|---|
| 2xx + valid shape | Response has the required fields | Format Success → return `{success: true, data}` |
| 2xx + malformed shape | Response is missing required fields | Format Failure - Malformed (non-retryable) |
| 4xx | Non-retryable client error | Format Failure - Not Retryable |
| 5xx / 429, attempts remaining | Retryable, under the 3-attempt bound | Wait 2s → retry |
| 5xx / 429 / connection error, attempts exhausted | Retryable, but 3 attempts used up | Format Failure - Retries Exhausted |

## Project layout

```
payment-link-generator-assignment/
├── assignment.pdf
├── README.md
├── PLAN.md
├── testcases.md
├── images/
│   ├── orchestrator-validation-dedup.png
│   ├── orchestrator-link-generation.png
│   ├── orchestrator-update-record.png
│   ├── subworkflow-generate-payment-link.png
│   └── subworkflow-update-payment-record.png
└── workflows/
    ├── generate-payment-link.json      # Sub-workflow: Payment Link API + retry
    ├── update-payment-record.json      # Sub-workflow: Update API + retry
    ├── notify-failure.json             # Shared Slack-alert sub-workflow
    └── payment-link-orchestrator.json  # Main workflow: intake → response
```

## Prerequisites

- A running n8n instance (self-hosted; Community plan is sufficient — this design deliberately avoids the paid-only Variables feature).
- Credentials configured in n8n (Settings → Credentials):
  - **Header Auth** credential for the Payment Link API (`token` header).
  - **Header Auth** credential for the Update API (`token` header).
  - **Header Auth** credential holding a random shared secret, used as the webhook's own auth.
  - **Google Sheets OAuth2** credential, for the audit log.
- A dedicated Google Sheet for the audit log, with a header row of exactly these 19 columns (order matters for `autoMapInputData`): `referenceId, validation_status, link_generation_status, record_update_status, final_status, payment_link, payment_id, invoiceNumber, customerName, customerEmail, customerPhone, contactId, paymentAmount, currency, description, expire_by, created_at, updated_at, error_detail`. The three `_status` columns are per-phase checkpoints, populated only once that phase is reached; `final_status` is the terminal/overall outcome used for dedup bucketing.
- A Slack Incoming Webhook URL for the alerting bonus feature.
- n8n started with two environment variables exported in its shell (see Setup & execution) — no hardcoded secrets anywhere in the workflow JSON.

## Setup & execution

*(Technical — skip ahead to Known Limitations if you're just here for the product story.)*

1. **Import the four workflow JSON files** into n8n (Workflows → Import from File), in any order.
2. **Wire credentials**: open each imported workflow and attach the credentials listed above to the relevant nodes (HTTP Request nodes for the two APIs, the Webhook trigger, and every Google Sheets node).
3. **Point the Google Sheets nodes** at your own audit spreadsheet's ID (Document dropdown/resource locator on each Sheets node).
4. **Point the `Execute Workflow` nodes** in the orchestrator at your own imported copies of `Generate Payment Link`, `Update Payment Record`, and `Notify Failure` (their workflow IDs will differ from this repo's).
5. **Start n8n with these environment variables exported first** — required because Slack's webhook URL is read via `$env` (n8n blocks env-var access in expressions by default, and paid-plan-only Variables isn't assumed to be available):
   ```bash
   export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
   export N8N_BLOCK_ENV_ACCESS_IN_NODE=false
   n8n start
   ```
6. **Activate all four workflows** (they call each other via Execute Workflow, which requires the target to be active/published).
7. **Test**: POST to the orchestrator's webhook URL with the documented payload shape and the shared-secret header.

### Happy path example

A fresh, valid request for a `referenceId` never seen before:

```bash
curl -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: <your shared secret>" \
  -d '{
    "referenceId": "TS1989",
    "contactId": "CONT_10001",
    "customerName": "Amit Sharma",
    "customerPhone": "+919876543210",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234",
    "paymentAmount": 1000,
    "currency": "INR",
    "description": "Payment for service order INV_10234"
  }'
```

What happens, step by step:
1. The webhook checks the shared-secret header, then validates the payload — all required fields are present and well-formed, so validation passes (`validation_status: validation_passed`).
2. The audit Sheet is checked for an existing row with `referenceId: TS1989` — none found, so this is treated as a new request.
3. `Generate Payment Link` calls the Payment Link API and returns a `payment_link`; the audit row is updated (`link_generation_status: link_generated`).
4. `Update Payment Record` calls the Update API with that link; the audit row is updated again (`record_update_status: update_confirmed`, `final_status: update_confirmed`).
5. The webhook responds once, synchronously, with a `200`:
   ```json
   {
     "success": true,
     "referenceId": "TS1989",
     "status": "Payment link generated and record updated",
     "data": { "payment_link": "https://api.xyz.com/v1/payment_links/123456" }
   }
   ```

More scenarios (validation errors, duplicate handling, gateway failures, malformed requests) are in `testcases.md` as ready-to-run curl commands.

## Known Limitations & Production Considerations

What's stubbed for this take-home versus what a production build would need:

- **The Google Sheets dedup/audit mechanism has a race window.** The dedup lookup and the eventual row write aren't atomic, so two near-simultaneous requests for the same `referenceId` could both pass the dedup check. Would need an atomic conditional write to fully close.
- **No async reconciliation sweep for process-level crashes.** Recovery is workflow-level (in-request retries) only. A crash inside a sub-workflow right after its external call actually succeeded, but before returning, would leave a permanently misleading audit row with no automatic correction.
- **`referenceId` is the sole idempotency key, with no mismatch-conflict check.** A resend of an existing `referenceId` with *different* field values (e.g. a changed `paymentAmount`) isn't detected as a conflict — the original stored `payment_link` is returned unconditionally and the new values are silently discarded.
- **Retry timing is a flat interval (3 attempts, 2s wait), not exponential backoff** — not evaluated against payment-gateway-specific backoff guidance.
- **`payment_link` is relayed to the customer with only a field-presence check**, not a URL-shape/domain validation.
- **Free-text fields (`customerName`, `description`) are only presence/shape-checked, not sanitized** — a leading `=`/`+`/`-`/`@` could in principle trigger spreadsheet formula execution if the audit Sheet is later opened in Excel/Sheets by an operator.
- **The audit Sheet's access/retention policy is undocumented** beyond "same Google account access as this n8n instance." No stated sharing boundary or deletion policy for the customer PII it accumulates (name, email, phone, amount).
- **No secrets are hardcoded** in any node parameter — API tokens, the webhook shared secret, and OAuth credentials are all n8n-managed credentials; the one exception (Slack's webhook URL) is an environment variable, never a hardcoded value in the workflow JSON. `error_detail` captures only response status/message, never request headers or credential values.

## Future Enhancements

- An async reconciliation sweep (scheduled companion workflow) to catch and correct process-level crashes the in-request retries can't cover.
- A stricter idempotency check that rejects a resend carrying different field values under the same `referenceId`, instead of silently discarding them.
