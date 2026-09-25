# Architecture decision records

Technology-stack decisions for Atlas, made 2026-08-29 against the spec set in [`docs/specs/`](../specs/). Framing for every decision: **production-grade from day one, ~1M patients onboarded, an existing client with real patient data to migrate**. No MVP-staging assumptions.

Format: one decision per file — Status, Context, Decision, Consequences, Alternatives rejected. A superseding decision gets a new numbered file; the old one is marked Superseded, never edited away.

| # | Decision |
|---|---|
| [0001](0001-react-native-expo-for-mobile.md) | React Native + Expo for the patient and doctor mobile apps |
| [0002](0002-nextjs-for-webapp-and-admin-console.md) | Next.js for the patient/doctor webapp and the admin console |
| [0003](0003-kotlin-spring-boot-services.md) | Kotlin + Spring Boot for backend services |
| [0004](0004-service-topology-six-deployables.md) | Six deployable services along spec boundaries — no monolith, no microservice sprawl |
| [0005](0005-hapi-fhir-interoperability.md) | HAPI FHIR for the interoperability and ingest layer |
| [0006](0006-kafka-async-backbone.md) | Kafka as the async backbone from day one |
| [0007](0007-data-stores.md) | Data stores: PostgreSQL (+RLS) primary, Redis, OpenSearch — nothing else |
| [0008](0008-keycloak-identity-provider.md) | Keycloak as the identity provider |
| [0009](0009-opa-policy-engine.md) | OPA as the policy engine for RBAC × ABAC × consent |
| [0010](0010-eks-terraform-mumbai.md) | EKS + Terraform, Mumbai region, per-tenant KMS encryption |
| [0011](0011-contract-first-apis.md) | Contract-first APIs with generated TypeScript clients |
