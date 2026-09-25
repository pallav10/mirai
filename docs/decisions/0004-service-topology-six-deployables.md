# 0004 — Six deployable services along spec boundaries

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [10-architecture](../specs/10-architecture.md) §9, [05-ai-features](../specs/05-ai-features.md), [11-integrations](../specs/11-integrations.md)

## Context

The architecture spec describes a service-oriented platform. Production framing rules out a single monolith (independent scaling and blast-radius isolation for real patient data), but nothing justifies fine-grained microservice sprawl: ~1M patients means low-hundreds RPS at peak on the hottest path.

## Decision

**Six deployable services**, boundaries matching the spec's domain seams:

| Service | Owns | Scaling driver |
|---|---|---|
| Identity & auth | Patient/practitioner/tenant identity, sessions, identity mapping | Sign-in bursts |
| Records & consent | Clinical record CRUD, patient-360 composition, consent grants | Read-heavy; replica-backed |
| QR & session | QR token lifecycle, scan resolution, consult sessions, break-glass | The signature latency path |
| Integrations | FHIR/HIS ingest pipeline, processing states, DLQ, provenance | Batch/migration load — the one that truly needs independent scale |
| AI gateway | Provider adapters, RAG over the authorized slice, generation logging | Provider latency isolation |
| Audit | Append-only audit stream consumer, query API, archive | Write throughput; must never backpressure the caller |

- Every cross-service sensitive action flows through the audit stream (ADR 0006); audit emission is fire-and-forget via outbox, never a synchronous dependency.
- Each service owns its schema; no shared database access across service boundaries (spec §13 store table maps stores to owners).

## Consequences

- Independent deploy and scale where it matters (integrations during client-data migration; audit under write load) without a 20-service operational tax.
- Service boundaries are contract-frozen via ADR 0011; splitting a service later (e.g. break-glass out of QR & session) is additive.
- Six on-call surfaces, six dashboards — the observability baseline (ADR 0010) is mandatory, not optional.

## Alternatives rejected

- **Modular monolith:** right for an MVP; rejected under the production framing — the integrations pipeline alone (bulk migration of the existing client's data) must scale and fail independently of the consult path.
- **Fine-grained microservices (per spec-section granularity):** operational cost with no scaling justification at this load.
