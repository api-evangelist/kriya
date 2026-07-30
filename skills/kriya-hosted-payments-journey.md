---
name: Take payment through the Kriya hosted Payments Journey
description: >-
  Use the Kriya-hosted checkout journey to collect buyer details, run onboarding and
  complete a PayLater payment with minimal integration, then reconcile the outcome.
api: openapi/kriya-payments-openapi.yaml
operations:
  - CreateOrder
  - GetOrder
  - UpdateOrder
  - TransitionOrderStatus
generated: '2026-07-19'
method: generated
source: https://docs.kriya.co/payments#section/Integration-Scenarios/Payments-Journey-Integration
---

# Take payment through the Kriya hosted Payments Journey

The Payments Journey is a Kriya-hosted web application that handles the checkout
flow. Kriya collects company and user details, runs onboarding where needed, and
returns the buyer to you. It is the lower-effort integration — roughly a week versus
one to four weeks for direct API — at the cost of the buyer leaving your domain and
of Kriya controlling the UX.

## Before you start

- Send `X-Kriya-ApiKey` on every request.
- Decide up front whether you want the order auto-transitioned on completion.

## Steps

1. **Create the order with journey fields.** Call `CreateOrder`, supplying the
   `paymentJourney` object. Two fields are required to get a redirect URL back:
   - `paymentAcceptedRedirectUrl` — where the buyer lands after completing.
   - `paymentDeclinedRedirectUrl` — where the buyer lands if they will not or cannot
     proceed.

   Optionally set `autoTransitionOnCompletion` to `Submitted` or `ReadyToAdvance`.
   By default a completed journey leaves the order in `Draft`, meaning you must
   transition it yourself in step 4. Only set `ReadyToAdvance` if you are certain —
   it releases funds irreversibly.

2. **Redirect the buyer.** The `201` response carries
   `paymentJourneyRedirectUrl`, valid for **14 days**. Redirect the buyer there. If
   the buyer is new, Kriya hands off to the Onboarding Journey automatically and
   hands control back when done — no extra work on your side. Returning buyers have
   their details pre-populated.

3. **Receive the buyer back.** Kriya redirects to your accepted or declined URL.
   Either way an order now exists, still in `Draft` unless you set
   `autoTransitionOnCompletion`.

4. **Reconcile the outcome.** Call `GetOrder` and read `paymentJourney.declineStatus`
   to find out what happened:
   - `UserInitiated` — the buyer chose to go back. You may amend the amount with
     `UpdateOrder` and redirect them into the journey again.
   - `Ineligible` — the buyer is not approved. Offer another payment method.
   - `InsufficientFunds` — the order exceeds their available limit. Offer another
     method, or take partial payment so Kriya funds the remainder.
   - `OnboardingChecksAreRequired` — a director must complete additional checks.
   - `OnboardingChecksAreFailed` — checks were attempted and failed; offer another
     method.
   - `InsufficientFundsOnboardingChecksAreRequired` — both apply.

5. **Submit the order.** If you did not use `autoTransitionOnCompletion`, call
   `TransitionOrderStatus` with `newStatus: Submitted` once the buyer places the
   order. From here the flow matches the direct-API skill from step 6 onward.

## Keeping state in sync

Do not rely on the redirect alone. Subscribe to the `OrderStatusChanged` webhook to
track the order beyond checkout — `Advanced`, `Overdue`, `Repaid`, `Closed`. Webhook
delivery is not guaranteed: Kriya retries three times with 2s/4s/8s backoff and then
stops, so also reconcile periodically by re-fetching orders. Check the event
`timestamp` and ignore events superseded by a more recent one.

Verify every webhook by recomputing a base64 HMAC-SHA256 digest of the raw payload
with your Kriya HMAC key and comparing it to the `X-Kriya-Signature` header. See
`asyncapi/kriya-payments-webhooks.yml`.

## Testing

Point at `https://api.kriya.dev/payments/` with your Test key. Use the Test-only
scenario simulators to force otherwise non-deterministic states —
`BuyerRiskDecisionScenario` to pin a buyer's decision status or limit, and
`TransitionOrderStatusScenario` to reach internal statuses like `Advanced` or
`Repaid`. Transitions must still be valid. Expect occasional 1-15 second responses in
Test due to cold starts. See `sandbox/kriya-sandbox.yml`.
