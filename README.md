# Payment Link Generation Workflow

## Overview

A modular n8n workflow that receives a payment request via webhook, generates a payment link through an external Payment Link API, and updates the payment record through an external Update API — with retry handling, deduplication, and an audit trail. As a bonus, beyond the assignment's stated scope, it also emails the customer their link and alerts Slack on unrecoverable failures.

## Planned architecture

- **Two independent sub-workflows**, each owning its own conditional-retry policy:
  - `Generate Payment Link` — calls the Payment Link API
  - `Update Payment Record` — calls the Update API
- **One orchestrator workflow** that:
  - receives the request over an authenticated webhook
  - validates the payload and checks a Google Sheets audit log for duplicates
  - calls both sub-workflows in sequence, logging every outcome
  - *(bonus, beyond assignment scope)* sends the customer a confirmation email exactly once
  - *(bonus, beyond assignment scope)* alerts Slack on unrecoverable failures
  - returns a single synchronous response

## Project layout

```
payment-link-generator-assignment/
├── assignment.pdf
├── README.md
└── workflows/
    ├── generate-payment-link.json
    ├── update-payment-record.json
    └── payment-link-orchestrator.json
```

Full architecture details, setup instructions, assumptions, and design rationale will be filled in once the workflows are complete.
