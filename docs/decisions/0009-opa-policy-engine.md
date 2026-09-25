# 0009 — OPA as the policy engine for RBAC × ABAC × consent

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [07-consent-and-access-control](../specs/07-consent-and-access-control.md), [10-architecture](../specs/10-architecture.md) §17, [06-identity-auth](../specs/06-identity-auth.md)

## Context

Every data read/write passes a policy decision after authentication — RBAC (role) × ABAC (tenant, facility) × consent (patient grants, QR-session scope, break-glass) per spec 07. The architecture spec names the class: "OPA/Cedar-class with consent extension." The decision point sits on the hot path (QR scan → 360° view), so evaluation must be local and fast, and every decision must be explainable for audit.

## Decision

**Open Policy Agent**, deployed as a sidecar next to each service (ADR 0004):

- Policies in Rego, versioned in the monorepo, tested in CI with a policy test suite covering spec 07's decision tables (including break-glass and revocation edge cases) before any deploy.
- **Consent as data, not code:** consent grants and QR-session scopes are pushed to the sidecars as OPA data documents via the event stream; policy logic stays static, decisions change with the data.
- Every decision returns an explanation payload (matched rules, consent grant IDs) that the calling service stamps into the audit event — "why was this access allowed" is answerable from the audit log alone.
- Local sidecar evaluation keeps policy checks off the network; decision latency budget ≤ 5 ms p99, **inclusive of the revocation-tombstone lookup below**.

## Revocation propagation — resolved

*(Resolved 2026-08-29, closing the open item in the original Consequences.)*

**The binding requirement is internal:** spec 07 §5.1 and its acceptance criterion — *a request by the revoked grantee within 1 second of revocation returns DENY*. Research found no stricter external bound: the DPDP Act requires consent withdrawal honored "without unnecessary delay" with near-real-time cascade across systems but sets no number; the ABDM consent framework mandates revocation notification to HIUs with no latency figure. Industry reference points show pure async propagation cannot meet 1 s worst-case: OPA bundle polling defaults to tens of seconds, and Zanzibar-class ACL replication quotes ~10 s bounded staleness. The 1 s bound therefore forces a synchronous element.

**Design: asymmetric two-path propagation.** Grants and revocations have opposite failure modes — a stale missing grant merely delays new access (fail-closed), a stale revoked grant is unauthorized PHI access (fail-dangerous) — so they take different paths:

- **Grants — async:** `consent.granted` flows Kafka → sidecar data documents. Staleness SLO p99 ≤ 5 s, paged at 30 s. Absence of a grant is DENY, so lag is never a safety issue.
- **Revocations — synchronous tombstone:** the revoke request path writes a tombstone to Redis (clustered, multi-AZ — ADR 0007) *before* returning success to the patient; the `consent.revoked` event still goes out via the outbox for durable propagation, cache invalidation, and notifications. Every consent-derived ALLOW performs one tombstone lookup (~1 ms intra-AZ, inside the 5 ms decision budget). Tombstones carry a 60 s TTL — async propagation removes the grant from sidecar data well within it.
- **Guarantee, not statistic:** the 1 s DENY bound is met by construction (the tombstone exists before the patient sees "Revoked"), not as a percentile.
- **Failure mode:** tombstone store unreachable → consent-derived decisions fail closed (DENY); RBAC/ABAC-only decisions are unaffected. Availability is sacrificed for revocation safety.
- **Measurement:** continuous synthetic canary per cell — revoke a canary grant, probe until DENY; end-to-end p99 ≤ 1 s paged. Grant-propagation staleness measured separately against its 5 s SLO.

## Consequences

- The revocation guarantee couples the consent hot path to Redis availability (fail-closed); the canary and the tombstone-store health become paged signals.
- The consent service owns two write paths (sync tombstone + async outbox) that must stay consistent; the 60 s tombstone TTL is the reconciliation window and is asserted in integration tests.
- Rego competence required in the platform team; policy review joins code review as a gate on anything touching access control.
- The policy suite becomes the executable form of spec 07 — divergence between spec tables and Rego tests is a build failure.

## Alternatives rejected

- **Cedar:** strong verification story, but a younger ecosystem, and its AWS-service coupling (Verified Permissions) pulls policy evaluation off-box and off the latency budget.
- **Policy logic in application code:** scatters spec 07 across six services; unexplainable, untestable as a unit, and guaranteed to drift.
- **Pure async propagation for revocations** (Kafka/bundle push only): cannot meet the 1 s worst-case DENY bound — ~10 s is the realistic staleness floor for replicated ACL data.
- **Synchronous push-with-ack to all sidecars on revoke:** meets the bound but couples the patient's revoke action to fleet health — one wedged sidecar blocks or falsifies the "Revoked" confirmation. The tombstone gives the same guarantee against one HA store instead of N sidecars.
- **Synchronous consent lookup on every decision** (no sidecar consent data at all): simplest consistency, but puts a network dependency on 100 % of decisions instead of only the tombstone check, and turns Redis into the availability ceiling for all reads.
