# 0010 — EKS + Terraform, Mumbai region, per-tenant KMS encryption

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [10-architecture](../specs/10-architecture.md) §13–17, [13-non-functional](../specs/13-non-functional.md)

## Context

Production launch with real patient data under Indian jurisdiction (DPDP Act, ABDM alignment) fixes residency; the architecture spec requires tenant isolation tiers (pooled/siloed at launch), per-tenant encryption keys at every tier, immutable audit, failover inside RTO/RPO, and OpenTelemetry observability. Six services, Kafka, Keycloak, and OPA sidecars (ADRs 0004, 0006, 0008, 0009) need an orchestration substrate from day one.

## Decision

- **AWS ap-south-1 (Mumbai)**, multi-AZ. No cross-border replicas, backups, or log shipping — residency is absolute, including derived data (search indexes, embeddings, telemetry).
- **EKS** runs all services, Keycloak, and OPA sidecars. Siloed-tier tenants get namespace + node-pool isolation now; the dedicated-cell tier later is new cells, not new service code (the architecture spec's acceptance criterion).
- **Terraform for everything** — VPC, EKS, RDS, MSK, OpenSearch, KMS, Keycloak realms, alarms. No console-created resources; environments (staging, prod) are the same modules with different variables.
- **Encryption:** per-tenant KMS keys, envelope encryption for PHI at rest in every store (Postgres, OpenSearch, S3, Kafka); TLS everywhere; mTLS service-to-service inside the mesh.
- **Audit immutability:** audit service writes to append-only storage; archives to S3 Object Lock (compliance mode). Nobody, including platform admins, can rewrite history.
- **Observability:** OpenTelemetry traces/metrics/logs from every service; the QR-scan→360° trace and the consent-revocation propagation lag (ADR 0009) are the two golden signals with paged SLOs.
- **Edge:** CloudFront + WAF in front of the gateway (Kong/Envoy-class per spec §17).

## Consequences

- Kubernetes operational competence is a day-one platform-team requirement; this is the cost of the tiered-isolation model.
- Game-day drills are scheduled work, not aspiration: regional AZ failover inside RTO/RPO, and the single-tenant point-in-time restore drill from the architecture spec's acceptance criteria.
- Staging environment carries synthetic personas only — never a copy of the client's patient data.

## Alternatives rejected

- **ECS/Cloud Run-class:** simpler, but per-tenant node isolation, sidecar patterns, and the cell model map poorly; migrating to K8s later with live tenants is the expensive path.
- **Multi-cloud abstraction:** cost without a requirement; residency and the cell model are cloud-agnostic in design (Terraform modules), which is enough optionality.
