# Payment Link Generation Workflow — Plan

*This reflects the plan as originally scoped, including one feature (the confirmation email) later cut during the build. See `README.md` for what was actually delivered.*

## Overview

A modular n8n workflow receives a payment request via webhook, generates a payment link through a mock Payment Link API, and updates the payment record through a dedicated sub-workflow — with conditional retry handling, deduplication by `referenceId`, and a full audit trail in Google Sheets. As a bonus, once the update is confirmed the workflow emails the customer their payment link, and pushes a Slack alert on unrecoverable failures.

## Architecture

Three n8n workflows: two independent, reusable sub-workflows and one orchestrator that ties them together.

```
Source system
   │ POST (shared-secret header)
   ▼
Main Orchestrator
   1. Webhook intake, header-auth rejects unauthenticated requests
   2. Validate required fields → 400 + audit log if missing
   3. Dedup lookup by referenceId against the audit Sheet
   4. Call Generate Payment Link sub-workflow → log result
   5. Call Update Payment Record sub-workflow → log result
   6. Send confirmation email once the update is confirmed
   7. Respond once, synchronously, with the final outcome
        │
        ├── Execute Workflow ──▶ Generate Payment Link
        │                        POST → Payment Link API
        │                        Retry: 5xx/timeout/connection only
        │
        └── Execute Workflow ──▶ Update Payment Record
                                 GET-with-body → Update API
                                 Retry: same policy, update-only
```

## Key Decisions

- **Modular sub-workflows, not one monolith.** `Generate Payment Link` and `Update Payment Record` each own their retry policy and are independently reusable, called from the orchestrator via Execute Workflow.
- **Synchronous response contract.** The webhook holds the connection open and returns one final response once both API calls settle, rather than acknowledging immediately and processing async.
- **Retry only transient failures.** 5xx, timeout, and connection errors retry (bounded attempts); a 4xx or malformed response fails fast, since retrying a permanently bad request just wastes attempts. Mirrors Stripe's/Razorpay's documented retry guidance.
- **Retry-the-update-only recovery.** If the Update API fails after a payment link was already generated, only the Update call is retried — the Payment Link API is never re-called, which would otherwise risk a second, orphaned link.
- **`referenceId` as the dedup key**, checked against the audit Sheet before any external call — mirrors the idempotency-key convention Stripe/Razorpay document.
- **Sheet is the audit trail; Slack is the push-alert channel** for unrecoverable failures only — routine cases (validation, duplicates) aren't alerted, to keep Slack signal-only.
- **Bonus: email notification**, sent exactly once per confirmed update, never once per retry attempt.
- **New credentials for every external call**, never hardcoded tokens in node parameters.

## Requirements (condensed)

- **Intake & validation** — receive the payload via webhook; validate required fields (`referenceId`, `paymentAmount`, `currency`, `customerEmail`, `invoiceNumber`) before calling any external API; a missing field fails fast with an audit log entry and no retry.
- **Duplicate detection** — before generating a link, check the audit Sheet for an existing completed row with the same `referenceId`; if found, return the existing link without re-calling the API.
- **Payment link generation** — call the Payment Link API; retry transient failures with a bounded attempt count; fail fast on non-transient ones.
- **Update & recovery** — call the Update API using the generated link; on failure, retry only the update, distinguishing an unrecoverable update failure from an unrecoverable link failure in the audit log.
- **Notification** — once the update is confirmed, email the customer their payment link, exactly once per confirmation.
- **Response contract** — hold the webhook connection open and return one synchronous JSON response with an appropriate status code.
- **Audit & credentials** — log every outcome (success, recovered, both failure types, duplicate, validation failure) to the audit Sheet; store every credential in n8n, never in a node parameter.
- **Security** — the webhook requires header authentication; an unauthenticated request is rejected before any workflow logic runs.
- **Operator alerting** — push a Slack notification for unrecoverable failures and sub-workflow contract violations only.

## Implementation Units

- **U3 — Generate Payment Link sub-workflow.** Encapsulates the Payment Link API call and its transient-only retry policy behind a typed, reusable interface.
- **U4 — Update Payment Record sub-workflow.** Encapsulates the Update API call and its retry policy; because it only ever runs after a link already exists, "retry only the update" falls out of the sub-workflow boundary for free.
- **U5 — Main orchestrator workflow.** Wires webhook intake, validation, dedup, both sub-workflow calls, Sheets logging, confirmation email, Slack alerting, and the synchronous response into one workflow.
- **U6 — Deliverable documentation.** The final README (architecture, setup, assumptions, design rationale) and the exported workflow JSON for all three workflows.
