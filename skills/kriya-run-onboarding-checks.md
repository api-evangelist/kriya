---
name: Run Kriya onboarding and KYC checks on a buyer company
description: >-
  Create a Kriya Onboarding Journey for a limited company, government entity or sole
  trader, run company credit checks plus optional identity, sanction and selfie
  checks, and track the resulting onboarding status.
api: openapi/kriya-onboarding-openapi.yaml
operations:
  - CreateOnboardingJourney
  - CreateBlankOnboardingJourney
  - UpdateOnboardingStatusScenario
generated: '2026-07-19'
method: generated
source: https://docs.kriya.co/onboarding#section/Integration-Scenarios
---

# Run Kriya onboarding and KYC checks

Onboarding is what a buyer completes before they can transact on Kriya. It always
includes required company checks, and may include additional checks the merchant opts
into: identity verification and proof of address, sanction screening, and a selfie
check.

## Before you start

- The Onboarding API uses the **same** `X-Kriya-ApiKey` credentials as the Payments
  API. No separate key is needed.
- Base URLs: `https://api.kriya.dev/onboarding/` for Test,
  `https://api.kriya.co/onboarding/` for Production.
- The API surface is deliberately small — its only job is to initiate the Onboarding
  Journey. The journey itself is a hosted web application; you do not drive the
  checks through the API.
- Additional checks for limited companies must be completed by a **registered
  director**. Sole traders complete the process themselves. Kriya can currently run
  limited-company additional checks only for UK-based companies.

## Steps

1. **Create the journey.** Call `CreateOnboardingJourney` with
   `merchantCompanyIdentifier`, optionally `kriyaCompanyIdentifier`, `nationalId`,
   `companyType`, `requesterEmail` and `requesterPhoneNumber`. Supply the
   `onboardingJourney` object to control where the buyer is sent afterwards:
   `completeUrl`, `ineligibleUrl`, `abandonedUrl` and `inReviewUrl`.

   The response returns `onboardingRedirectUrl`. Redirect the buyer there.

   Use `CreateBlankOnboardingJourney` instead when you have no company details yet
   and want Kriya to collect everything.

   As an alternative to the API, you may redirect the buyer straight to
   `https://api.kriya.co/onboardingJourney/create` with a `merchantIdentifier` query
   parameter (required) plus optional `merchantCompanyIdentifier`,
   `onboardingCompletedUrl` and `onboardingIneligibleUrl`. Kriya responds with a
   Location header to the new journey. Undefined completion URLs default to
   `https://kriya.co`.

2. **Understand the identifier binding.** Each `merchantCompanyIdentifier` is
   permanently bound to exactly one `kriyaCompanyIdentifier` on first association.
   Once set it cannot be changed, and passing the same merchant identifier later
   auto-fills the company details. Generate these carefully — a mistake is not
   reversible through the API.

3. **Route the director.** After company checks complete and a director is chosen,
   the director can continue in three ways: proceed directly if they are the current
   user, receive a copied link passed through any channel, or follow an email
   invitation Kriya sends to the director's address.

4. **Track the outcome.** The Onboarding API does not expose a status read
   operation. Track completion through the Payments API instead — the
   `BuyerRiskDecisionChanged` webhook carries an `onboardingStatus` field, and
   `GetSingleBuyer` returns current buyer status. A buyer with onboarding status
   `Approved` is a verified buyer; anything else is unverified. Until additional
   checks pass, non-draft order placement for that company is blocked.

## Where this fits with payments

If you use the hosted Payments Journey, onboarding is embedded automatically — Kriya
hands off to the Onboarding Journey and back, and you need no separate integration.
You only need this skill when you are integrating directly against the API, or when
you want to onboard buyers ahead of any purchase.

Two direct-API cases require this mixed approach:
- **Sole traders**, which direct API cannot create at all.
- **Merchants requiring additional checks** where no director has yet passed them —
  the order flow stalls until the director completes onboarding.

## Testing

In the Test environment, call `UpdateOnboardingStatusScenario` to force a company's
onboarding status rather than waiting on real checks. It is Test-only — never call it
against Production. See `sandbox/kriya-sandbox.yml`.

## Handling errors

Onboarding errors return a `problemDetails` body that adds `metadata`, `reasons` and
`statusCode` to the standard envelope, with each entry in `errors` carrying a `code`
and `message`. Kriya does not publish the set of possible codes. See
`errors/kriya-problem-types.yml`.
