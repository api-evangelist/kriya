---
name: Onboard a Kriya buyer and place a B2B PayLater order
description: >-
  Take a business buyer from company lookup through Kriya risk decisioning to a
  submitted buy-now-pay-later order, using the Kriya Payments API direct integration.
api: openapi/kriya-payments-openapi.yaml
operations:
  - CompaniesSearch
  - CreateBuyer
  - GetSingleBuyer
  - FindBuyerPricingScheme
  - CreateOrder
  - TransitionOrderStatus
generated: '2026-07-19'
method: generated
source: https://docs.kriya.co/payments#section/Integration-Scenarios/Direct-API-Integration
---

# Onboard a Kriya buyer and place a PayLater order

Kriya lets a merchant offer trade credit to business buyers. A buyer must be
registered and approved before any order can be submitted against their spending
limit.

## Before you start

- Send `X-Kriya-ApiKey` on every request. Test and Production keys are different and
  are not interchangeable. See `authentication/kriya-authentication.yml`.
- Work against `https://api.kriya.dev/payments/` while integrating; Production is
  `https://api.kriya.co/payments/`.
- There is no idempotency-key contract. Do not blind-retry writes — re-read state
  first with `GetOrder` or `GetSingleBuyer`.
- Monetary amounts are int64 minor units with an ISO 4217 currency of `GBP`, `USD` or
  `EUR`, matching the buyer's country of registration.

## Steps

1. **Resolve the company.** Call `CompaniesSearch` to obtain a
   `kriyaCompanyIdentifier`. Prefer `countryCode` + `nationalId` — that combination
   usually returns a single exact match. Fall back to name + address, or URL/email,
   only when a national-id search returns nothing. The US and Canada have no
   nationwide registry, so name + address or URL/email is the only option there.
   This endpoint is limited to **5 requests per second** — far tighter than the API
   default. Throttle accordingly.

2. **Register the buyer.** Call `CreateBuyer` with the `kriyaCompanyIdentifier`,
   `countryCode`, company `type` and `contactDetails`. A `201` confirms creation and
   starts automated risk decisioning. Note that direct API integration does not
   support sole-trader creation — route sole traders through the Onboarding Journey
   instead.

3. **Wait for the risk decision.** Do not poll tightly. Subscribe to the
   `BuyerRiskDecisionChanged` webhook (see
   `asyncapi/kriya-payments-webhooks.yml`), or call `GetSingleBuyer` on a modest
   interval. Proceed only on `status: Approved` with a non-zero `availableLimit`.
   - `Submitted` — decision pending, keep waiting.
   - `InReview` — manual review by the Kriya risk team; may take considerably longer.
   - `OnHold` — no new orders, though existing ones may still be transitioned.
   - `Rejected` — offer another payment method. `rejectionReasons` are for the
     merchant only and **must not be shown to the buyer**.

4. **Retrieve payment options.** Call `FindBuyerPricingScheme` to get the payment
   methods available to this buyer and their `feePercentage`. Fees are charged on top
   of the order amount and do not consume the buyer's spending limit — a £1,000 order
   on a 1% method costs the buyer £1,010 but reduces their limit by £1,000.

5. **Create the order.** Call `CreateOrder`. Orders default to `Draft`, which is
   non-committal and does not touch the spending limit. You may create `Draft` orders
   while a risk decision is still pending.

6. **Submit the order.** Call `TransitionOrderStatus` with `newStatus: Submitted`
   once the buyer confirms. This reduces the buyer's available spending limit by the
   order total, so the available limit must be greater than or equal to the PayLater
   portion. `Payment`, `User`, `BuyerCompany` and `SupplierCompany` (where
   applicable) must all be present by this point.
   - If the merchant requires additional onboarding checks and no director has passed
     them, this call will not succeed — send the director through the Onboarding
     Journey (see the companion skill) and resume afterwards.
   - If the merchant requires 2FA, the response returns a `SessionStatusUrl`. Poll
     `GetOrderMfaSession` or wait on the webhook. The session expires after **5
     minutes**; if no decision arrives, re-issue the request.

7. **Release funds after delivery.** Once goods are delivered and invoiced, call
   `TransitionOrderStatus` with `newStatus: ReadyToAdvance`.
   **This is irreversible.** It instructs Kriya to advance funds to the merchant and
   supplier, the order can no longer be cancelled, and no further updates are
   accepted. Require explicit human confirmation before an agent issues this call.

## Handling errors

- `400` — inspect the `errors` collection in the `problemDetails` body and fix the
  offending fields. The envelope is not RFC 9457.
- `401` — missing or invalid `X-Kriya-ApiKey`, or a Test key sent to Production.
- `409` — usually an invalid state transition. Re-read the order with `GetOrder` and
  confirm the transition is legal; for example `ReadyToAdvance` cannot go straight to
  `Repaid`.
- `429` — back off until the window refreshes. Sustained breaches can get API access
  disabled.

Full catalog: `errors/kriya-problem-types.yml` and `errors/kriya-decline-codes.yml`.
