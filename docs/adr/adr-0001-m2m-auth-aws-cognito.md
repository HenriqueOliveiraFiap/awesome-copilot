---
title: 'ADR-0001: M2M Authentication and Authorization Using AWS Cognito'
status: 'Proposed'
date: '2026-04-07'
authors:
  - 'Platform Engineering Team'
tags:
  - 'authentication'
  - 'authorization'
  - 'm2m'
  - 'aws-cognito'
  - 'oauth2'
  - 'microservices'
  - 'security'
supersedes: 'ADR-v2: M2M Authentication Strategy'
superseded_by: ''
---

# ADR-0001: M2M Authentication and Authorization Using AWS Cognito

## Status

**Proposed** — Supersedes ADR-v2: M2M Authentication Strategy

---

## Context

### Background

Our microservices architecture has grown from a small set of internally communicating services (v2) to a distributed multi-team, multi-service ecosystem where backend services communicate with one another without human user involvement. The v2 approach relied on static API keys and a basic OAuth 2.0 client credentials flow without a managed identity provider, which introduced the following pain points:

- **No centralised identity lifecycle management**: API keys were provisioned and rotated manually, creating operational and security gaps.
- **No fine-grained scoping**: Every service had access to every API; least-privilege was not enforced.
- **No auditability**: Token issuance and usage were not observable through standardised tooling.
- **Scaling and compliance pressures**: SOC 2 and ISO 27001 audit requirements mandate demonstrable access control, secret rotation, and audit logging for service-to-service communication.

### Why Cognito?

> **Critical thinking note**: Cognito User Pools are primarily designed for human (federated) identity. For M2M, we use Cognito **App Clients** with the OAuth 2.0 **Client Credentials Grant** — a pattern that bypasses user-facing flows entirely. This is a deliberate and supported use of Cognito for M2M, but teams should be aware that Cognito is not a purpose-built M2M identity provider (cf. Auth0, HashiCorp Vault, AWS IAM Roles Anywhere).

AWS Cognito is selected due to:

1. **Existing AWS infrastructure**: The organisation is AWS-native; Cognito integrates with API Gateway, CloudWatch, Secrets Manager, and IAM out of the box, reducing operational overhead.
2. **Managed JWKS endpoint**: Cognito exposes a well-known JWKS endpoint, enabling stateless, local JWT validation by each service without a runtime call to Cognito per request.
3. **Custom resource server scopes**: Cognito supports defining resource servers with custom OAuth scopes (e.g., `payments/process`, `api/read`), enabling least-privilege access control.
4. **Secrets Manager integration**: Client secrets can be stored and auto-rotated in AWS Secrets Manager, addressing the manual rotation gap in v2.

### What Changed from v2

| Dimension | v2 | v3 (this ADR) |
|---|---|---|
| Auth mechanism | Static API keys / ad hoc OAuth | OAuth 2.0 Client Credentials via Cognito App Clients |
| Identity provider | None (self-managed) | AWS Cognito User Pool (M2M App Clients) |
| Scope enforcement | None | Custom OAuth scopes per resource server |
| Secret management | Manual rotation | AWS Secrets Manager + Lambda auto-rotation (90-day cycle) |
| Token validation | Centralised gateway check | Local JWKS validation per service |
| Observability | Ad hoc logs | CloudWatch metrics + alarms |
| Perimeter enforcement | Application-level | API Gateway with Cognito Authorizer |

---

## Decision

**We will use AWS Cognito OAuth 2.0 Client Credentials Grant with Cognito User Pool App Clients as the M2M identity provider for all service-to-service authentication.**

### Key Design Choices

- **Per-service App Clients**: Each service acting as an OAuth client is issued its own Cognito App Client with a unique `client_id` and `client_secret`. This enables per-client auditing, revocation, and scope assignment.

- **Resource Servers and Custom Scopes**: Resource servers are registered in Cognito with fine-grained OAuth scopes (e.g., `api/read`, `api/write`, `payments/process`). Each App Client is granted only the scopes it requires (least-privilege).

- **Local JWT Validation via JWKS**: Receiving services validate JWTs locally by fetching and caching Cognito's JWKS endpoint (`https://cognito-idp.<region>.amazonaws.com/<userPoolId>/.well-known/jwks.json`). This eliminates a synchronous Cognito call on every request and avoids hitting Cognito's token rate limits (default: 10 token operations/second/User Pool).

- **Token TTL and Caching Strategy**: Access tokens are issued with a TTL of **300 seconds (5 minutes)**. Client services cache tokens in-memory and proactively refresh them **30 seconds before expiry**. For high-throughput services, a distributed cache (e.g., ElastiCache Redis) is used as a second cache tier to avoid thundering-herd token refresh storms.

- **Client Secret Storage and Rotation**: Client secrets are stored exclusively in **AWS Secrets Manager**. A Lambda function triggers automatic rotation every **90 days**, updating both Secrets Manager and the corresponding Cognito App Client atomically.

- **API Gateway as Perimeter Enforcer**: All inter-service HTTP traffic passes through API Gateway, which is configured with a **Cognito Authorizer**. This provides a consistent, infrastructure-level enforcement point before requests reach service code.

- **Observability**: CloudWatch custom metrics track token issuance rate, token validation failures, and scope usage per client. Alarms are set on anomalous patterns (e.g., sudden spike in `invalid_grant` errors, unusual scope requests).

### Architecture Diagram (Logical)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Service A (Client)                       │
│  1. Fetch client_secret from Secrets Manager (cached)           │
│  2. POST /oauth2/token → Cognito (Client Credentials Grant)     │
│  3. Cache JWT (in-memory, refresh 30s before TTL)               │
│  4. Attach Bearer token to request                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS + JWT
                            ▼
              ┌─────────────────────────┐
              │      API Gateway        │
              │  Cognito Authorizer     │  ← validates JWT signature
              │  (scope enforcement)    │    and required scopes
              └────────────┬────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Service B (Resource)                      │
│  5. Validate JWT locally via cached JWKS                        │
│  6. Assert required scope (e.g., payments/process)              │
│  7. Process request                                             │
└─────────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │      AWS Cognito        │
              │  User Pool + App Client │
              │  JWKS Endpoint          │
              │  Resource Servers       │
              └─────────────────────────┘
```

---

## Consequences

### Positive

- **POS-1 — Least-Privilege Enforcement**: Custom OAuth scopes per resource server allow fine-grained, auditable access control. Services cannot access resources outside their declared scope.

- **POS-2 — Stateless JWT Validation**: Local JWKS-based JWT validation eliminates synchronous Cognito calls per request, improving latency and resilience to Cognito availability events.

- **POS-3 — Automated Secret Rotation**: AWS Secrets Manager + Lambda rotation removes manual secret management, reducing the risk of stale or leaked credentials.

- **POS-4 — Auditability and Observability**: CloudWatch integration provides a centralised view of token issuance, failures, and scope usage, satisfying audit requirements (SOC 2, ISO 27001).

- **POS-5 — Native AWS Integration**: Cognito integrates natively with API Gateway, CloudWatch, Secrets Manager, and IAM, reducing the operational surface area for a team already invested in AWS.

- **POS-6 — Perimeter Enforcement at Infrastructure Layer**: API Gateway with Cognito Authorizer enforces authentication and scope checks before requests reach application code, providing defence-in-depth.

### Negative

- **NEG-1 — Cognito Rate Limits**: Cognito imposes a default limit of **10 token operations per second per User Pool**. This can become a bottleneck for high-throughput microservices. Mitigation: aggressive token caching (in-memory + Redis) and proactive refresh minimise token issuance frequency. Service quota increases must be requested proactively.

- **NEG-2 — No Native mTLS**: Cognito does not support mutual TLS (mTLS) for M2M authentication. JWT bearer tokens are the only supported credential type. For environments requiring mTLS (e.g., PCI-DSS tier), an alternative such as AWS Private CA with API Gateway mTLS must be layered on top.

- **NEG-3 — Manual Client Secret Rotation Complexity**: While Secrets Manager automates rotation, the Lambda rotation function must atomically update both Secrets Manager and the Cognito App Client. Any failure in this two-step process can leave credentials in an inconsistent state. The rotation Lambda must be idempotent and include rollback logic.

- **NEG-4 — Vendor Lock-in**: Adopting Cognito as the M2M identity plane creates a dependency on the AWS ecosystem. Migrating to a multi-cloud or cloud-agnostic identity provider (e.g., HashiCorp Vault, Auth0) in the future would require re-architecting token issuance and potentially updating all client services.

- **NEG-5 — Limited Scope Flexibility Compared to Purpose-Built M2M Providers**: Cognito's resource server / scope model is less flexible than providers like Auth0 or Okta. Dynamic scope policies, fine-grained RBAC, and per-request claim enrichment are not natively supported without custom Lambda authorizers or post-token-generation triggers.

- **NEG-6 — Debugging Complexity**: Diagnosing M2M authentication failures in a distributed system requires correlating Cognito logs, API Gateway access logs, and application-level JWT validation errors across CloudWatch Log Groups. This is more complex than a centralised identity management dashboard.

- **NEG-7 — Cost at Scale**: Cognito charges for MAUs (Monthly Active Users), but for M2M App Clients, the cost model is based on token operations. At high token issuance volumes, costs must be modelled explicitly. Token caching directly reduces cost as well as latency.

---

## Alternatives Considered

- **ALT-1 — Retain v2 Static API Keys**: Simple and low-overhead, but provides no scope enforcement, no automated rotation, no auditability, and does not scale to a multi-team microservices environment. Rejected due to compliance and operational maturity requirements.

- **ALT-2 — Auth0 / Okta for M2M**: Purpose-built M2M identity providers with richer scope models, built-in secret rotation, mTLS support, and better developer experience for M2M flows. The principal objection is that the organisation is deeply AWS-native; introducing a third-party SaaS identity provider adds an external dependency, increases cost, and requires additional integration work. Revisit if multi-cloud strategy is adopted.

- **ALT-3 — AWS IAM Roles with SigV4 (Service-to-Service)**: For AWS-internal service communication, IAM roles with SigV4 request signing is a strong alternative that avoids token issuance entirely. Rejected for this use case because: (a) not all services are AWS-hosted (some are containerised workloads with no IAM role), and (b) SigV4 is AWS-specific and would not work for services outside the AWS boundary. Recommended as a complementary pattern for purely AWS-internal, same-account service calls.

- **ALT-4 — HashiCorp Vault with JWT/OIDC Auth**: Vault provides a flexible, cloud-agnostic secret and identity management platform. Strong for multi-cloud or hybrid environments. Rejected at this stage due to operational overhead of running and maintaining a Vault cluster and the team's existing AWS expertise. Viable future path if multi-cloud adoption increases.

- **ALT-5 — AWS IAM Roles Anywhere (mTLS + X.509)**: Supports certificate-based M2M identity for workloads outside AWS. Addresses the mTLS gap in this ADR. Rejected as primary mechanism due to complexity of PKI management, but noted as a complementary control for high-assurance service communication in PCI-DSS scope.

---

## Implementation Notes

- **IMP-1 — Cognito User Pool Configuration**: Create a dedicated User Pool for M2M (separate from any human-facing User Pool). Enable the OAuth 2.0 authorization server domain. Register resource servers with custom scopes.

- **IMP-2 — App Client Provisioning**: Provision one App Client per service. Disable all grant types except `client_credentials`. Assign only the minimum required scopes. Store `client_id` and `client_secret` in Secrets Manager under a consistent naming convention (e.g., `cognito/m2m/<service-name>/client-secret`).

- **IMP-3 — Token Caching Implementation**: Implement a two-tier cache: (1) in-process cache (e.g., a thread-safe singleton) with expiry set to `token_expiry - 30s`; (2) distributed cache (ElastiCache Redis) for services with multiple replicas to prevent thundering-herd token refresh. Use a mutex/lock on token refresh to prevent concurrent refresh storms.

- **IMP-4 — JWKS Caching**: Cache the Cognito JWKS response locally with a TTL of 1 hour. Implement a fallback to re-fetch JWKS on JWT validation failure (key rotation event). Do not fetch JWKS on every request.

- **IMP-5 — Secrets Manager Rotation Lambda**: The rotation Lambda must implement the four-step rotation lifecycle: `createSecret`, `setSecret`, `testSecret`, `finishSecret`. The `setSecret` step must update the Cognito App Client secret atomically. Include CloudWatch alarms on rotation failure.

- **IMP-6 — API Gateway Cognito Authorizer**: Configure the Cognito Authorizer on API Gateway with the User Pool ARN. Specify required scopes per route. Set authorizer result TTL to 0 for M2M flows (tokens are short-lived and cached client-side; API Gateway caching of authorizer results can cause stale scope enforcement).

- **IMP-7 — CloudWatch Observability**: Publish custom metrics: `TokenIssuanceRate` (per client), `TokenValidationFailures` (per service), `ScopeViolationAttempts`. Create a CloudWatch Dashboard for M2M auth health. Set alarms on: `TokenValidationFailures > threshold`, `RotationFailures > 0`, `TokenIssuanceRate > 80% of Cognito quota`.

- **IMP-8 — Zero Trust Enforcement**: Each service must validate the JWT independently (do not trust API Gateway's authorization decision alone). Validate: signature (JWKS), `iss` (Cognito issuer URL), `aud` (resource server identifier), `exp` (not expired), and required scopes.

- **IMP-9 — Incident Response Runbook**: Document the process for revoking a compromised App Client (rotate secret in Cognito + Secrets Manager, invalidate any cached tokens, alert downstream services). Revoked tokens remain valid until expiry (300s TTL); plan for this window in incident response.

---

## References

- **REF-1** — [AWS Cognito OAuth 2.0 Client Credentials Grant](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html)
- **REF-2** — [AWS Cognito Resource Servers and Custom Scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- **REF-3** — [AWS Secrets Manager Secret Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- **REF-4** — [AWS API Gateway Cognito Authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html)
- **REF-5** — [IETF RFC 6749: OAuth 2.0 Client Credentials Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)
- **REF-6** — [AWS Cognito Service Quotas](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html)
- **REF-7** — [AWS Well-Architected Framework — Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- **REF-8** — [NIST SP 800-207: Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- **REF-9** — [AWS IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html)
- **REF-10** — ADR-v2: M2M Authentication Strategy *(internal, superseded by this document)*
