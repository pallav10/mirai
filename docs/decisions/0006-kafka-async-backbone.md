# 0006 — Kafka as the async backbone from day one

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [10-architecture](../specs/10-architecture.md) §12, [11-integrations](../specs/11-integrations.md), [13-non-functional](../specs/13-non-functional.md)

## Context

Three workloads need durable async messaging at production launch: the integration ingest pipeline with processing states and DLQ (spec 11), the audit event stream that every sensitive action feeds (spec §13 — append-only, must never drop events), and AI generation triggers (spec 05, async consumers). The existing client's data migration puts real bulk volume on the pipeline immediately — there is no low-volume grace period in which a lighter queue would suffice.

## Decision

**Apache Kafka** (managed — MSK) is the single async backbone:

- Topics per domain stream: `integration.inbound`, `integration.deadletter`, `audit.events`, `ai.generation`, `notifications.outbound`.
- Producers use the **transactional outbox pattern** — domain write and event emission commit atomically in Postgres, a relay publishes to Kafka. No dual-write anywhere.
- `audit.events` is the compliance artery: replicated, long retention, consumed by the audit service into append-only storage with object-lock archive (ADR 0010).
- DLQ + replay tooling for the integration pipeline is built with the pipeline, not after — the admin console's error queue (spec 04) fronts it.

## Consequences

- Ordered, replayable history for audit and ingest — reprocessing a bad migration batch is a replay, not a re-extract from the client.
- Kafka operational weight is real; taking it managed (MSK) and day-one keeps one messaging system for the platform's life instead of a BullMQ→Kafka migration mid-flight with patient data in the pipe.
- Consumer lag on `audit.events` becomes a paged alert — audit falling behind is a compliance incident, not a performance footnote.

## Alternatives rejected

- **BullMQ/Redis then migrate to Kafka:** the MVP path. Rejected — the migration would land exactly when the integration pipeline is busiest, and Redis-backed queues give neither replay nor ordered retention for audit.
- **SQS/SNS:** simpler ops, but no replay, weak ordering, and per-topic fan-out sprawl; audit replay-ability alone disqualifies it.
