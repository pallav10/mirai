# 0003 — Kotlin + Spring Boot for backend services

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [10-architecture](../specs/10-architecture.md) §17, [11-integrations](../specs/11-integrations.md), [12-api-and-interoperability](../specs/12-api-and-interoperability.md)

## Context

The architecture spec (§17) suggests "Kotlin/Java or Go microservices; gRPC internal, REST external" as the service-tier technology class. The production framing — real patient data from an existing client, FHIR-aligned ingest, long-lived compliance-heavy services, hiring in India — decides between them. An all-TypeScript backend (NestJS) was considered for type-sharing with the four React client surfaces and rejected.

## Decision

Backend services are **Kotlin on Spring Boot** (JVM):

- gRPC between services, REST (and FHIR, ADR 0005) at the external boundary — as the architecture spec prescribes.
- Spring Security + the OPA sidecar (ADR 0009) at every service edge; the gateway-signed tenant context is verified in a shared library, never re-derived per service.
- One shared Kotlin platform library: tenant context, audit event emission, outbox publishing, OTel instrumentation.

## Rationale

1. **FHIR is the decider.** HAPI FHIR — the only battle-tested FHIR engine — is Java. A JVM service tier consumes it natively (ADR 0005); any other language rebuilds FHIR validation, terminology, and resource handling by hand.
2. **Ecosystem maturity for the compliance load:** Spring Security, mature Kafka clients, OTel, gRPC — all first-class on the JVM.
3. **Hiring:** the India healthcare-backend talent pool is deepest on JVM languages.
4. **Type-sharing with clients survives without language coupling** — contract-first codegen (ADR 0011) replaces it, and is the correct production mechanism anyway.

## Consequences

- Two-language stack (TypeScript clients, Kotlin services); the API contract (ADR 0011) is the sole coupling point and must be treated as a first-class artifact.
- JVM operational competence (GC tuning, container memory sizing) required in the platform team.

## Alternatives rejected

- **Go:** leaner runtime, but weak FHIR ecosystem and a narrower healthcare-domain hiring pool. The one trade worth reopening if the backend team arrives Go-native.
- **NestJS/TypeScript:** wins on type-sharing and single-language velocity — an MVP argument. Loses FHIR tooling, and the codegen contract makes the sharing argument moot at production scale.
