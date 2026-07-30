# Kriya

Kriya (Kriya Finance Limited, London) is a UK B2B embedded-finance and working-capital
provider that lets merchants offer trade credit to their business buyers — Embedded
PayLater, Invoice Finance, working capital loans, Buyer Authentication, Offline
Payments and Kriya on Stripe. Kriya was acquired by Allica Bank in October 2025.

Website: https://www.kriya.co — Developer docs: https://docs.kriya.co/

Backed by: northzone

## APIs

| API | Docs | Base URL | Spec |
|---|---|---|---|
| Kriya Payments API | https://docs.kriya.co/payments | `https://api.kriya.co/payments/` | [openapi/kriya-payments-openapi.yaml](openapi/kriya-payments-openapi.yaml) |
| Kriya Onboarding API | https://docs.kriya.co/onboarding | `https://api.kriya.co/onboarding/` | [openapi/kriya-onboarding-openapi.yaml](openapi/kriya-onboarding-openapi.yaml) |

Both OpenAPI 3.0.1 documents were harvested from `https://cdn.kriya.co/images/`,
which backs the Redoc portal at docs.kriya.co. 20 operations, 49 schemas.

## Artifacts

| Artifact | File | Method |
|---|---|---|
| Authentication | [authentication/](authentication/kriya-authentication.yml) | searched |
| Conventions | [conventions/](conventions/kriya-conventions.yml) | searched |
| Webhooks | [asyncapi/](asyncapi/kriya-payments-webhooks.yml) | searched |
| Sandbox | [sandbox/](sandbox/kriya-sandbox.yml) | searched |
| Embedded components | [components/](components/kriya-components.yml) | searched |
| Decline codes | [errors/](errors/kriya-decline-codes.yml) | searched |
| Error catalog | [errors/](errors/kriya-problem-types.yml) | derived |
| Data model | [data-model/](data-model/kriya-data-model.yml) | derived |
| Conformance | [conformance/](conformance/kriya-conformance.yml) | derived |
| Lifecycle | [lifecycle/](lifecycle/kriya-lifecycle.yml) | searched |
| Overlays | [overlays/](overlays/) | generated |
| Agent skills | [skills/](skills/_index.yml) | generated |
| llms.txt | [llms/](llms/kriya-llms.txt) | generated |
| MCP (candidate) | [mcp/](mcp/kriya-mcp.yml) | derived |
| Domain security | [security/](security/kriya-domain-security.yml) | probed |
| Vulnerability disclosure | [security/](security/kriya-vulnerability-disclosure.yml) | searched |
| Well-known (none found) | [well-known/](well-known/kriya-well-known.yml) | searched |
| Packages (none found) | [packages/](packages/kriya-packages.yml) | searched |

## Integration notes

- Auth is a static partner API key in the `X-Kriya-ApiKey` header. Neither spec
  declares `securitySchemes` — the overlays add it.
- No OAuth, no scopes, and **no idempotency contract**.
- Rate limits: 2000 req / 5 min per key; company search 5 req/sec.
- Errors use a `problemDetails` envelope that is **not** RFC 9457.
- Four HMAC-signed webhook events (`X-Kriya-Signature`, HMAC-SHA256, base64), 3
  retries with 2s/4s/8s backoff, delivery not guaranteed.
- A real Test environment at `api.kriya.dev` with Test-only `/scenario/` simulators.
- `TransitionOrderStatus` → `ReadyToAdvance` advances real funds and is irreversible.

## Not published

No `/.well-known/` documents, status page, changelog, deprecation or versioning
policy, SLA, AsyncAPI document, MCP server, CLI, GitHub organisation, or any
first-party SDK in npm/PyPI. Packages named "kriya" in the public registries belong
to unrelated projects.
