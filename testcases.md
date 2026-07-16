# Test Cases — Payment Link Orchestrator Webhook

Every case below is a ready-to-run `curl` command — copy, paste, run locally.

## One-time setup

```bash
export WEBHOOK_SECRET="paste-your-secret-here"   # n8n → Settings → Credentials → "Payment Webhook - Shared Secret"
```

Do this once per terminal session. The commands below all reference `$WEBHOOK_SECRET` — **never hardcode the real secret into this file**, since it's committed to the repo.

- **`referenceId` values below are placeholders.** Each one should be unique per test run — the workflow is idempotent per `referenceId`, so re-running a case with the same `referenceId` hits the *duplicate-handling* logic (Section 5) instead of re-running the original path. Suffix them with a run number/timestamp if you re-run this file (e.g. `TC-101-HAPPY-FULL-2`).
- Every response is checked against the sheet in the actual build — if you want to confirm the audit trail too, check the "Payment Link Audit Log (n8n)" sheet after each call. The sheet tracks each phase separately: `validation_status`, `link_generation_status`, and `record_update_status` are checkpoint columns populated only once that phase is reached (blank if never attempted), and `final_status` is the terminal/overall outcome.

---

## Section 1 — Happy path

### TC-1.1: Fresh valid request, all fields
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-101-HAPPY-FULL",
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
**Expected**: `200 OK`
```json
{
  "success": true,
  "referenceId": "TC-101-HAPPY-FULL",
  "status": "Payment link generated and record updated",
  "data": { "payment_link": "https://api.xyz.com/v1/payment_links/123456" }
}
```

### TC-1.2: Valid request, optional fields omitted (only the 5 mandatory fields)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-102-HAPPY-MINIMAL",
    "paymentAmount": 500,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10235"
  }'
```
**Expected**: `200 OK`, same shape as TC-1.1.

---

## Section 2 — Authentication

### TC-2.1: Missing `X-Webhook-Secret` header
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -d '{
    "referenceId": "TC-201-NOAUTH",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `403 Forbidden`, plain text (not JSON, not our envelope — n8n's native Header Auth rejects before any workflow logic runs):
```
Authorization data is wrong!
```
Response has a `WWW-Authenticate: Basic realm="Webhook"` header and no `Content-Type: application/json` — confirmed live against this exact build. This is the documented known-gap case: the auth rejection deliberately isn't reshaped into our envelope, since doing so would mean moving auth out of n8n's native Header Auth (which rejects before any workflow logic runs) and into a manual in-workflow check, weakening that guarantee.

### TC-2.2: Wrong `X-Webhook-Secret` value
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: wrong-secret-value" \
  -d '{
    "referenceId": "TC-202-BADAUTH",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `403 Forbidden`, same as TC-2.1.

---

## Section 3 — Validation errors

Every case here should short-circuit at `Validate Input` — no Sheets write beyond the failure row, no Payment Link API call, no Update API call. Each one also fires a Slack alert (`:rotating_light: *Validation Failed*` with the reference ID and a bulleted list of failing fields) — check your Slack channel alongside the HTTP response.

### TC-3.1: Missing a mandatory field (`referenceId`)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400 Bad Request`
```json
{
  "success": false,
  "referenceId": null,
  "status": "Validation failed",
  "data": {
    "stage": "validation",
    "error": { "code": "VALIDATION_ERROR", "category": "client_error", "retryable": false, "message": "One or more fields failed validation" },
    "fields": [ { "field": "referenceId", "reason": "missing" } ]
  }
}
```

### TC-3.2: Missing multiple mandatory fields at once
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-302-MULTI-MISSING",
    "customerEmail": "amit.sharma@example.com"
  }'
```
**Expected**: `400`, `data.fields` contains entries for `paymentAmount`, `currency`, and `invoiceNumber`, each `reason: "missing"`.

### TC-3.3: `paymentAmount` wrong type (string instead of number)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-303-AMOUNT-STRING",
    "paymentAmount": "1000",
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `fields: [{ "field": "paymentAmount", "reason": "wrong_type" }]`

### TC-3.4: `paymentAmount` zero or negative
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-304-AMOUNT-NEGATIVE",
    "paymentAmount": -50,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `fields: [{ "field": "paymentAmount", "reason": "must_be_positive" }]`

### TC-3.5: `currency` wrong format (lowercase)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-305-CURRENCY-LOWERCASE",
    "paymentAmount": 1000,
    "currency": "inr",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `fields: [{ "field": "currency", "reason": "invalid_format" }]`

### TC-3.6: `currency` wrong length
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-306-CURRENCY-LENGTH",
    "paymentAmount": 1000,
    "currency": "INDIA",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `fields: [{ "field": "currency", "reason": "invalid_format" }]`

### TC-3.7: `customerEmail` invalid format
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-307-EMAIL-INVALID",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "not-an-email",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `fields: [{ "field": "customerEmail", "reason": "invalid_format" }]`

### TC-3.8: `customerPhone` invalid format (optional field, present but malformed)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-308-PHONE-INVALID",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234",
    "customerPhone": "not-a-phone-number"
  }'
```
**Expected**: `400`, `fields: [{ "field": "customerPhone", "reason": "invalid_format" }]`

### TC-3.9: Several validation failures in one request
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-309-MULTI-INVALID",
    "paymentAmount": -5,
    "currency": "inr",
    "customerEmail": "not-an-email",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected**: `400`, `data.fields` contains **all three**: `paymentAmount` (`must_be_positive`), `currency` (`invalid_format`), `customerEmail` (`invalid_format`) — proves the validator collects every failure, not just the first.

---

## Section 4 — `expireBy` (optional field, its own validation)

### TC-4.1: No `expireBy` provided (default expiry used)
Same as TC-1.1 — just confirm via the sheet: `expire_by` column should show a value ≈ (now + 7 days) in Unix seconds.

### TC-4.2: Valid `expireBy` override (1 hour from now)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d "{
    \"referenceId\": \"TC-402-EXPIREBY-VALID\",
    \"paymentAmount\": 1000,
    \"currency\": \"INR\",
    \"customerEmail\": \"amit.sharma@example.com\",
    \"invoiceNumber\": \"INV_10234\",
    \"expireBy\": $(( $(date +%s) + 3600 ))
  }"
```
**Expected**: `200 OK`, success. Sheet's `expire_by` column shows the computed timestamp exactly, not the 7-day default.

### TC-4.3: `expireBy` in the past
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-403-EXPIREBY-PAST",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234",
    "expireBy": 1000000000
  }'
```
**Expected**: `400`, `fields: [{ "field": "expireBy", "reason": "in_past" }]`

### TC-4.4: `expireBy` exceeds the 3-year cap
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d "{
    \"referenceId\": \"TC-404-EXPIREBY-MAX\",
    \"paymentAmount\": 1000,
    \"currency\": \"INR\",
    \"customerEmail\": \"amit.sharma@example.com\",
    \"invoiceNumber\": \"INV_10234\",
    \"expireBy\": $(( $(date +%s) + 4*365*24*60*60 ))
  }"
```
**Expected**: `400`, `fields: [{ "field": "expireBy", "reason": "exceeds_max" }]`

### TC-4.5: `expireBy` wrong type (string instead of number)
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-405-EXPIREBY-STRING",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234",
    "expireBy": "tomorrow"
  }'
```
**Expected**: `400`, `fields: [{ "field": "expireBy", "reason": "wrong_type" }]`

---

## Section 5 — Duplicate / idempotency handling

These require **two sequential calls with the same `referenceId`**.

### TC-5.1: True duplicate (resend after full success)
```bash
# Call 1 — expect normal success
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-501-DUP-CONFIRMED",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'

# Call 2 — same referenceId, expect "Already processed"
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-501-DUP-CONFIRMED",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected on call 2**: `200 OK`, fast response (no Payment Link API / Update API calls happen):
```json
{
  "success": true,
  "referenceId": "TC-501-DUP-CONFIRMED",
  "status": "Already processed",
  "data": { "payment_link": "https://api.xyz.com/v1/payment_links/123456" }
}
```

### TC-5.2: Resend after a validation failure (should fully reprocess, not treated as duplicate)
```bash
# Call 1 — invalid amount, expect 400
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-502-RETRY-AFTER-VALIDATION",
    "paymentAmount": -50,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'

# Call 2 — same referenceId, now valid, expect full success
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-502-RETRY-AFTER-VALIDATION",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected on call 2**: `200 OK`, full success — proves a validation failure doesn't "lock" the `referenceId`.

### TC-5.3: Resend after `update_unrecoverable` (retry-the-update-only recovery)
This one needs a manual setup step since it can't be forced purely by the request body — see **Section 7** for how to break `Update Payment Record`'s endpoint temporarily via n8n.

```bash
# Step 1 — with the Update API endpoint broken, expect 502 (RECORD_UPDATE_* code)
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-503-RETRY-UPDATE-ONLY",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'

# Step 2 — restore the Update API endpoint in n8n, then resend the same referenceId
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{
    "referenceId": "TC-503-RETRY-UPDATE-ONLY",
    "paymentAmount": 1000,
    "currency": "INR",
    "customerEmail": "amit.sharma@example.com",
    "invoiceNumber": "INV_10234"
  }'
```
**Expected on step 2**: `200 OK`, full success — and (checkable via n8n execution history, not from curl output) `Generate Payment Link` was **not** called again, only `Update Payment Record`.

---

## Section 6 — Malformed request body

### TC-6.1: Invalid JSON syntax
```bash
curl -s -i -X POST http://localhost:5678/webhook/payment-link \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: $WEBHOOK_SECRET" \
  -d '{"referenceId": "TC-601-BROKEN-JSON", "paymentAmount": 1000,'
```
**Expected**: `422 Unprocessable Entity`, n8n's own body-parser error (not our envelope) — this happens before the webhook trigger fires at all:
```json
{"code":422,"message":"Failed to parse request body", "hint": "...", "stacktrace": "..."}
```
**Known issue** (documented, not fixed): this response leaks internal file paths in `stacktrace`. Not fixable at the workflow level; would need a reverse proxy in front of n8n to sanitize it.

---

## Section 7 — Gateway / backend failures

**These cannot be triggered by the request body alone** — they require temporarily pointing the relevant sub-workflow's HTTP node at a failing test endpoint (via the n8n UI or MCP tools), running the curl command, then restoring the real URL. Documented here for completeness/reproducibility, not as pure curl cases.

| Scenario | How to force it | Expected response `error.code` |
|---|---|---|
| Payment Link API rejects (4xx) | Point `Generate Payment Link` → `Call Payment Link API` at `https://httpbin.org/status/400` | `GATEWAY_REJECTED` |
| Payment Link API returns malformed 2xx | Point at an endpoint returning `200` with an empty/unexpected body | `GATEWAY_CONTRACT_VIOLATION` |
| Payment Link API unreachable (connection-level) | Point at an unresolvable domain | `GATEWAY_TIMEOUT` |
| Payment Link API repeatedly 5xx/429 | Point at `https://httpbin.org/status/503` | `GATEWAY_UNAVAILABLE` |
| Update API rejects / malformed / unreachable / 5xx | Same four variants, on `Update Payment Record` → `Call Update API` | Same four codes, `stage: "record_update"`, plus `data.payment_link` populated |
| Google Sheets write fails (any write node) | Point the relevant Sheets node's `documentId` at an invalid/inaccessible sheet ID | `INTERNAL_ERROR`, `stage: "audit_log"` |
| Genuine unexpected crash (any Code node) | Temporarily insert `throw new Error('test')` at the top of any Code node's `jsCode` | `UNEXPECTED_ERROR`, `stage: "unknown"` |

All seven were exercised and verified live during this build. **Remember to restore the original URL/code immediately after testing.**

Every failure path in this table also fires a Slack alert via the shared `Notify Failure` sub-workflow — not just validation and gateway failures. The Sheets-unavailable path sends `:rotating_light: *Audit Log Unavailable*` with the reference ID; the unexpected-crash catch-all sends `:rotating_light: *Unexpected Internal Error*` with the reference ID (or `unknown` if the crash happened before `Validate Input` finished) and the error message. Both were confirmed live via forced-failure tests.

---

## Quick reference — all `error.code` values

| Code | Stage | Retryable | Meaning |
|---|---|---|---|
| `VALIDATION_ERROR` | `validation` | false | Input failed existence/type/format checks |
| `INTERNAL_ERROR` | `audit_log` | true | Google Sheets unavailable |
| `GATEWAY_REJECTED` | `link_generation` / `record_update` | false | Gateway returned a definitive 4xx |
| `GATEWAY_CONTRACT_VIOLATION` | `link_generation` / `record_update` | false | Gateway returned 2xx but an unusable body |
| `GATEWAY_TIMEOUT` | `link_generation` / `record_update` | true | Connection-level failure after 3 attempts |
| `GATEWAY_UNAVAILABLE` | `link_generation` / `record_update` | true | Repeated 5xx/429 after 3 attempts |
| `SUBWORKFLOW_CONTRACT_VIOLATION` | `link_generation` / `record_update` | false | Our own sub-workflow returned an unexpected shape |
| `UNEXPECTED_ERROR` | `unknown` | false | Genuinely unanticipated crash, caught by the workflow-wide fallback |
