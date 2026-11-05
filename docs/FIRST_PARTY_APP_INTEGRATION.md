# First-party application integrations

This gateway is scoped to two first-party applications:

- MAIRA AI VOICE AGENT (`maira-ai-voice-agent`)
- SMART MONEY (`smart-money`)

Each application has its own merchant record, API key hash, encrypted webhook secret, webhook endpoint and receiving-account configuration. Do not share credentials between the applications.

## Provisioning

Configure these variables only on the payment API server (Vercel project environment or private server environment). Never put them in Vite variables, browser JavaScript, public HTML, Git commits or customer-facing app settings.

| Variable | Purpose |
| --- | --- |
| `MAIRA_GATEWAY_API_KEY` | Server-to-server order creation key for MAIRA |
| `MAIRA_GATEWAY_WEBHOOK_SECRET` | HMAC secret used to sign MAIRA webhook events |
| `MAIRA_GATEWAY_WEBHOOK_URL` | HTTPS webhook receiver deployed by MAIRA |
| `SMART_MONEY_GATEWAY_API_KEY` | Server-to-server order creation key for SMART MONEY |
| `SMART_MONEY_GATEWAY_WEBHOOK_SECRET` | HMAC secret used to sign SMART MONEY webhook events |
| `SMART_MONEY_GATEWAY_WEBHOOK_URL` | HTTPS webhook receiver deployed by SMART MONEY |

Provide all three variables for an app or none. A partial set stops API startup with a configuration error. Use independently generated random values (API key at least 40 characters; webhook secret at least 32 characters). The API hashes API keys and encrypts webhook secrets at rest. Startup creates or updates the two fixed merchant identities idempotently; it never prints credential values.

After provisioning, use the admin dashboard to configure the receiving UPI account for each merchant identity. Do not reuse an account unless the receiving bank has explicitly approved the intended collection use. The checkout QR uses the receiving account assigned to the merchant on the order.

## Create an order (server-to-server only)

The app backend must authenticate its own customer first, calculate the amount from its own trusted product/package catalogue, and then call:

`POST /api/v1/payments`

Headers:

- `Content-Type: application/json`
- `X-Merchant-Id: <merchant UUID>`
- `X-Merchant-Key: <server-only app API key>`
- `Idempotency-Key: <unique app-side order attempt key>`

Body example:

```json
{
  "orderId": "MAIRA-COIN-ORDER-123",
  "amount": "99.00",
  "currency": "INR",
  "returnUrl": "https://your-app.example/payment/pending",
  "metadata": {
    "productType": "COIN_PACK",
    "productCode": "starter",
    "userId": "app-user-id"
  }
}
```

For SMART MONEY, use a package identifier and immutable user reference in metadata instead. The app server must not accept amount, package price, coins or user identity as trusted values from a browser request.

The response includes `paymentId` and `checkoutUrl`. Redirect the customer to the hosted checkout. The browser must never receive either app's merchant API key or webhook secret.

Production order creation rejects anonymous/default-merchant requests. A valid merchant API key or correctly signed order token is required. Local development can retain the demo fallback unless `REQUIRE_MERCHANT_API_KEY=true` is set.

## Webhook contract and fulfillment rule

Webhook headers include:

- `X-Gateway-Event`
- `X-Gateway-Signature: sha256=<HMAC-SHA256(raw request body)>`
- `X-Gateway-Signature-V2: sha256=<HMAC-SHA256(timestamp + "." + raw request body)>`
- `X-Gateway-Timestamp`
- `X-Gateway-Delivery-Id`

Verify the HMAC against the exact raw request bytes with a constant-time comparison. Also verify the timestamp freshness, event name, expected merchant/order/payment ID, exact amount, INR currency, product metadata and unique delivery ID. Persist processed delivery IDs with a unique database constraint before acknowledging the request.

| Event | Meaning | App action |
| --- | --- | --- |
| `OBSERVED_SUCCESS` | Observer alert was attributed; not bank-settled | Keep order pending; do not credit coins or activate a package |
| `MANUAL_REVIEW` | Conflicting or ambiguous evidence | Hold order for admin review |
| `RECONCILED` | Official bank statement line matched the order | Verify all app-side order fields, then fulfill once |
| `SUCCESS` | COD/counter business event, not UPI bank settlement | Do not use for UPI fulfillment |
| `FAILED`, `EXPIRED`, `CANCELLED` | Non-payment terminal state | Update order state; do not fulfill |

The `RECONCILED` event is written to the durable outbox in the same database transaction as the status transition. Recovery scans can recreate a missing delivery for an already reconciled payment. Webhooks are at-least-once: the app receiver must be idempotent.

## Required app-side fulfillment

### MAIRA AI VOICE AGENT

- Preserve the existing Razorpay endpoints and checkout path as the rollback option.
- Add a server-side gateway order endpoint that authenticates the Supabase user, reads the pack price/coin count from a server-owned allow-list, creates a pending purchase row, and calls the gateway using the MAIRA credentials.
- Add a raw-body webhook endpoint that verifies both gateway signatures and freshness.
- On `RECONCILED` only, call a service-role-only, transaction-safe database function that validates the stored gateway order, user, pack, amount and currency, then inserts one unique ledger credit and marks the purchase paid atomically.
- A duplicate webhook, changed amount, wrong user, wrong order, stale signature or non-`RECONCILED` event must never credit coins.

### SMART MONEY

- Preserve the existing Razorpay endpoints and checkout path as the rollback option.
- Add a server-side gateway order endpoint that authenticates the Supabase user and derives the package price and user identity from trusted database records, not browser-supplied values.
- Add a raw-body webhook endpoint with the same HMAC, timestamp, order-binding and delivery-deduplication checks.
- On `RECONCILED` only, invoke a service-role-only transaction that records the verified payment and moves the matching package into the existing admin-approval workflow. Do not bypass the package's existing admin activation rules.
- Duplicate or mismatched events must not create duplicate package purchases, commissions or referral credits.

The gateway repository does not have access to either application's private production credentials or database state. These app-side receivers and atomic wallet/package database functions must be implemented and tested in their respective repositories before setting the webhook URLs above. Do not set those URLs to endpoints that have not yet been deployed.

## Receiving-bank statement operation

The admin dashboard's **Bank Reconciliation** view accepts a CSV downloaded from the receiving bank. It requires a receiving account selection and shows reconciled, conflicting and unmatched rows. A CSV import is an operator action, not an automatic bank API integration. SMS, email, screenshots, QR return redirects and customer "payment done" clicks are not settlement proof.

## Production boundary

This repository remains experimental until the bank/acquirer has approved the actual collection model, required legal/compliance obligations are met, production bank access is authorized, app-side fulfillment tests pass, and an independent security review is complete. Do not switch either app's live payment traffic or remove Razorpay as a fallback merely because the gateway checkout works in a sandbox.
