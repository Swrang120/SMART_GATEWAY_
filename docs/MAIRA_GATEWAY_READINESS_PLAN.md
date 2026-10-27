# SMART_GATEWAY_ — MAIRA AI VOICE AGENT Integration & Readiness Plan

Last reviewed: 2026-10-27

## Current verified state

- This repository is a multi-package Node.js workspace with separate payment API, merchant-admin, checkout portal, database/core packages, and observer apps.
- The root README describes the gateway as an experimental research/learning project and explicitly says it is not certified, bank-approved, NPCI-approved, audited, or production-ready.
- The existing security audit identifies unresolved real-money risks: bank settlement reconciliation, payment attribution edge cases, webhook operations, SSRF/egress controls, secrets rotation, privacy/OTP handling, compliance, backups and recovery.
- MAIRA AI VOICE AGENT currently has Razorpay order creation and verification routes. Its current payment flow should be preserved until the replacement passes the test plan.
- The root `index.html` added earlier is a static dashboard preview only. It is not the merchant-admin application and does not process payments.

## Required dashboard modules

1. Overview: date range, captured/reconciled totals, pending/failed/refunded counts, settlement status, recent activity and system health. All totals must be sourced from the database/API; no fabricated sample values in live mode.
2. Transactions: server-side pagination, filters, search, order/payment IDs, amount/currency, timestamps, status history, reconciliation state, export and safe detail view.
3. Receiving bank accounts: add/edit/deactivate, masked account identifiers, verification status, account ownership/authorization evidence, active/inactive state, audit history. Never store bank passwords, UPI PINs, OTPs or internet-banking credentials.
4. Payment links/orders: create, expire/cancel, amount, currency, customer reference, return URL allow-list, status, and idempotency key.
5. Checkout: branded customer checkout, clear amount/merchant identity, accessible mobile layout, explicit pending/failure/success states; success only after authoritative server verification.
6. API keys: create/rotate/revoke, key scopes, one-time secret display, hashed-at-rest storage where possible, audit logs and rate limits. Never expose server secrets to frontend.
7. Webhooks: HTTPS allow-list policy, HMAC signatures, timestamp/replay protection, unique delivery ID, retries with backoff, delivery log, manual replay, idempotent merchant receiver examples.
8. Refunds/disputes: show status and provider reference. Do not claim a refund was executed unless a real provider/bank operation confirms it.
9. Reports: date-range reports, CSV export, settlement reconciliation, fees/tax fields only when sourced from actual records.
10. Team/security: roles, MFA where supported, session expiry, login audit, least-privilege access, account recovery and security settings.
11. Settings: business details, webhook endpoints, allowed origins/return URLs, notifications, timezone, branding and test/live environment separation.
12. Support/operations: incident contact, health checks, audit trail, queue/backlog status, failed-job visibility, backup/restore and alerting.

## Safe implementation order

- Keep the current Razorpay flow intact and available as the rollback path.
- First run the existing build/typecheck/test suite and record failures before changes.
- Build a dedicated integration adapter in MAIRA, rather than scattering gateway-specific logic through the voice-agent UI.
- Add a gateway sandbox/test mode with synthetic payments only.
- Enforce server-side authentication and merchant/order/amount/currency matching on every payment verification.
- Add idempotency and database uniqueness constraints so retries cannot credit a wallet twice.
- Confirm webhook signature, freshness, expected order, amount, currency and deduplication before fulfilling a purchase.
- Add reconciliation states. An observed bank notification is not equivalent to confirmed settlement.
- Run automated unit, integration, replay, duplicate-event, timeout, restart and rollback tests.
- Only then perform a controlled provider/bank-approved pilot and independent security/compliance review.

## External requirements — owner action needed

These cannot be granted by code or by changing a UI:
- Written confirmation from the receiving bank/acquirer that the intended collection flow and account use are permitted.
- Any required payment-system, aggregator, merchant, UPI, KYC/AML, consumer-protection, privacy and tax approvals/obligations, based on the actual business model and advice from qualified professionals.
- Production bank/acquirer credentials and approved settlement/reconciliation access, if available.
- Verified business identity, customer support/refund/dispute process, terms, privacy policy, data retention and incident contact.
- Independent security review and production approval.
- Production secrets entered directly into the hosting provider's encrypted environment settings, never committed to Git or shared in chat.

## Explicit non-goals / safety limits

- This project must not imitate Razorpay's brand, proprietary UI assets or imply affiliation. It can offer familiar payment-dashboard patterns with original branding.
- Do not bypass bank/UPI rules or use personal-account notifications as a substitute for authoritative settlement confirmation.
- Do not represent the system as zero-fee, certified, or production-ready without verified evidence.
- Do not switch MAIRA live payments to this gateway merely because the UI looks complete.

## Acceptance checklist

- [ ] Root/dashboard route renders the actual application, not README content.
- [ ] Mobile and desktop navigation works; empty/loading/error states are designed.
- [ ] Dashboard figures come from authenticated API responses.
- [ ] No secret or personal/bank data appears in client bundles or logs.
- [ ] Orders have unique IDs and idempotency protection.
- [ ] Payment is fulfilled only after authoritative verification.
- [ ] Duplicate webhook/payment callbacks do not double-credit.
- [ ] Delayed, missing and contradictory bank observations go to pending/manual review.
- [ ] Refund and reconciliation states are accurate.
- [ ] Build, typecheck, automated tests, dependency audit and restore test pass.
- [ ] Test environment is separate from live credentials and live bank accounts.
- [ ] Bank/acquirer and legal/compliance approvals are documented before live launch.

## Important status

This document records repository inspection and an implementation checklist. It does not claim that the gateway has passed tests or is approved for real-money production. The earlier static `index.html` preview is not evidence of a working payment service.
