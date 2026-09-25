# 0008 — Keycloak as the identity provider

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [06-identity-auth](../specs/06-identity-auth.md), [07-consent-and-access-control](../specs/07-consent-and-access-control.md)

## Context

The identity spec sets a two-tier model: patients are platform-global identities authenticating via OTP (later biometrics/passkeys on device), while practitioners are tenant-scoped and authenticate through their organization's identity provider. Production launch includes a real tenant whose doctors need federated sign-in, and every auth event feeds the audit stream.

## Decision

**Keycloak** (self-hosted on EKS, per ADR 0010) is the platform IdP:

- **One patient realm** — platform-global. OTP via SMS provider as primary factor; passkey/biometric enrollment per spec 06. Patient identity never lives inside a tenant realm.
- **Per-tenant practitioner realms** — one Keycloak realm per tenant organization, each brokering that hospital's SAML/OIDC IdP, with local accounts as fallback for tenants without their own IdP. *(Resolved 2026-08-29; the single-realm-with-federation option is rejected.)* Rationale: a realm is a hard isolation boundary that matches the platform's tenancy model — per-tenant session policies, token lifetimes, password/MFA policies, and IdP configuration without cross-tenant coupling; a realm misconfiguration's blast radius is one tenant; tenant offboarding is realm deletion. Tenant count is hospitals/organizations — hundreds at most, well inside Keycloak's comfortable realm range; realm creation is a Terraform module invoked by tenant onboarding.
- Keycloak issues the tokens; the **API gateway derives and signs the tenant context** from them (spec 06 §5). Services trust only the signed context, never raw client claims.
- Keycloak event listener publishes sign-in success/failure and session termination to the audit stream (ADR 0006).

## Consequences

- Self-hosting means owning Keycloak upgrades, HA, and realm-config-as-code (Terraform provider) — the price of data residency and per-tenant federation flexibility.
- Tenant onboarding gains an identity step: broker the tenant's IdP, map roles, verify with a game-day sign-in test.
- Break-glass authentication (spec 06) is an Atlas flow layered on Keycloak sessions, not a Keycloak feature — it lives in the identity service.

## Alternatives rejected

- **Auth0/Okta:** per-MAU pricing is hostile at 1M patients, and PHI-adjacent identity data leaves the residency boundary.
- **AWS Cognito:** weak multi-tenant IdP brokering and inflexible OTP flows; fighting it costs more than running Keycloak.
- **Build fully custom:** OTP + sessions is buildable, but federated SAML/OIDC brokering for hospital IdPs is exactly the wheel not to reinvent.
