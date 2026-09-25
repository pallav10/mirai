# 0011 — Contract-first APIs with generated TypeScript clients

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [12-api-and-interoperability](../specs/12-api-and-interoperability.md), [09-data-model](../specs/09-data-model.md)

## Context

The stack is deliberately two-language: TypeScript on all four client surfaces (ADRs 0001, 0002), Kotlin on the services (ADR 0003). The single coupling point is the API. Drift between client and server understanding of the FHIR-aligned model, consent semantics, or audit payloads is a correctness bug in a clinical product — it must be made impossible, not reviewed for.

## Decision

**The contract is the source of truth; all bindings are generated. Nothing is hand-written twice.**

- **External APIs (gateway):** OpenAPI specs, versioned in the monorepo under `contracts/`, one spec per gateway surface (`IPatientAPI`, `IDoctorAPI`, `IAdminAPI` per spec 12). Kotlin server stubs and TypeScript clients are both generated in CI.
- **Internal APIs:** protobuf/gRPC definitions in the same `contracts/` tree; Kotlin bindings generated per service.
- **Event schemas:** Kafka topic payloads (ADR 0006) defined in the same tree with schema-registry enforcement — audit events and integration states included.
- **Breaking-change gate in CI:** contract diffs run a compatibility check; a breaking change fails the build unless the version is bumped and a migration note added. Additive-only evolution within a major version.
- Generated TS clients ship as monorepo packages (`@atlas/api-patient`, `@atlas/api-doctor`, `@atlas/api-admin`) consumed by mobile, web, and admin.

## Consequences

- The type-sharing benefit that argued for an all-TypeScript stack is recovered without language coupling.
- Contract review becomes a first-class gate: changes to `contracts/` require both a client-side and service-side reviewer.
- Mock servers generated from the same contracts let the four client teams build against surfaces before services land — the parallelization mechanism for a multi-team build.

## Alternatives rejected

- **Hand-written clients per surface:** four clients × every endpoint × every change; drift is guaranteed.
- **Code-first with generated docs (annotations → OpenAPI):** contract shape becomes an accident of server implementation and breaking changes are discovered downstream instead of failing the diff gate.
