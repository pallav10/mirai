# 0007 — Data stores: PostgreSQL (+RLS) primary, Redis, OpenSearch

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [10-architecture](../specs/10-architecture.md) §13, [09-data-model](../specs/09-data-model.md), [05-ai-features](../specs/05-ai-features.md) §7, [08-qr-subsystem](../specs/08-qr-subsystem.md)

## Context

The architecture spec fixes the technology classes: PostgreSQL with row-level security for clinical and identity data, Redis for session/token state, search + k-NN for record search and embeddings. At ~1M patients (≈20–50k DAU, low-hundreds RPS peak on the hottest path) this is well inside a partitioned Postgres deployment — the decisions here are configuration and discipline, not exotic scale technology.

## Decision

Exactly three store technologies; adding a fourth requires a new ADR.

**PostgreSQL** (managed, Mumbai region — ADR 0010) — system of record:
- Clinical DB partitioned by tenant; **RLS enforced for the pooled tier**, separate schemas/clusters for the siloed tier, per the spec's tiering table.
- Read replicas serve patient-360 composition; writes stay on primary.
- PgBouncer in front of every service pool.
- Per-service schemas with no cross-service access (ADR 0004); identity registry is its own platform-global database.

**Redis** (managed, clustered) — QR token state, consult sessions, OTP/rate-limit counters, and consent-revocation tombstones (the synchronous fast path that meets spec 07's 1 s revocation bound — ADR 0009). Nothing in Redis is the only durable copy of anything, but the tombstone path is load-bearing for revocation safety: multi-AZ, and consent-derived policy decisions fail closed if it is unreachable.

**OpenSearch** — record/timeline search **and** embedding storage via k-NN for the RAG pipeline (spec 05 §7). Index content derived from Postgres via the event stream; rebuildable from scratch.

## Consequences

- pgvector is **dropped**: OpenSearch is already in the stack for search, so k-NN there avoids a fourth store and keeps embeddings out of the clinical DB's backup/restore path.
- RLS is defense-in-depth behind the gateway-signed tenant context (spec 06 §5), not the sole isolation mechanism — both are mandatory, and the per-tenant restore drill in the architecture spec's acceptance criteria is the proof.
- Search/embedding indexes hold derived PHI — they inherit the same encryption, residency, and deletion obligations as the primary store; a tenant offboarding deletes their index content too.

## Alternatives rejected

- **pgvector for embeddings:** fine technology; rejected to hold the store count at three and keep vector workload off the clinical primary.
- **Citus/sharding:** unjustified at this load; tenant partitioning + replicas has an order of magnitude of headroom.
- **DynamoDB/Mongo for any clinical data:** the model is relational and consent evaluation is join-shaped; RLS is load-bearing in the isolation story.
