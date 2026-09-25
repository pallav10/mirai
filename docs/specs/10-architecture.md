# 10 — System architecture

End-to-end technical architecture for the Atlas platform: system context, layers, multi-tenancy model, core services, key flows, deployment, and the MVP boundary.

Sources: `Canvas.dc.html` (high-level system design diagram), `Atlas Architecture.dc.html` (end-to-end architecture document, §1–§20), `Atlas LLD.dc.html` (component diagram tab; sequence tabs for flow detail), `Atlas_Project_Requirements.md` §28 (API-first), §31 (non-functional), §32 (suggested high-level architecture), §33 (core design principle).

Related specs: component-level interfaces and diagrams in [Atlas LLD](../../design/Atlas%20LLD.dc.html); API detail in [12-api-and-interoperability.md](12-api-and-interoperability.md); data entities in [09-data-model.md](09-data-model.md); AI product behavior in [05-ai-features.md](05-ai-features.md); auth in [06-identity-auth.md](06-identity-auth.md); consent in [07-consent-and-access-control.md](07-consent-and-access-control.md); QR mechanics in [08-qr-subsystem.md](08-qr-subsystem.md); integrations in [11-integrations.md](11-integrations.md); NFR detail in [13-non-functional.md](13-non-functional.md).

## 1. System context and goals

Atlas is a multi-tenant, patient-centric health-record platform: one patient → one identity → one QR → complete medical history → controlled doctor access. It aggregates a patient's medical history across healthcare encounters and exposes it, under patient control, to authenticated healthcare professionals through a QR-based access mechanism. Hospitals keep their existing EHR/HIS/LIS as systems of record; Atlas is the patient-centric aggregation and access layer on top.

Context definitions:

- **Tenant** = a healthcare organization (hospital, clinic chain, diagnostics network). Tenants own the clinical records they produce.
- **Patient identity is global** — one Atlas Patient ID (e.g. `ATL-82X92K`) spans tenants, so a consolidated timeline can be composed across organizations with consent. Patients are platform-level identities, not tenant members.
- **Doctors belong to tenants** and authenticate through their organization's identity provider.
- External actors: hospital EHR/HIS/LIS systems (push finalized results via the integration layer only, never write to core stores) and AI model providers (reached only through the internal provider gateway).

Primary quality goals, in priority order: security and auditability first; near-instant QR-to-patient lookup; high availability during active patient care; interoperability-ready data model (FHIR-aligned).

The PRD §32 sketch (Patient app presenting a QR; Doctor web/mobile → Atlas API → AuthN/AuthZ → {Patient & Identity, Medical Records, Audit & Access Logs} → Document Storage) is the ancestor of this design; every box in that sketch maps to a layer or service below, elaborated for multi-tenancy, integration, eventing, and AI.

### Acceptance criteria

- An architecture reviewer can trace each PRD §32 element to a concrete layer/service in this document.
- No component outside the integration layer accepts writes from external hospital systems.
- The design supports the PRD §33 principle verbatim: "The QR code identifies the patient; authentication and authorization determine what the doctor can see."

## 2. Architecture principles

These eight principles (Architecture doc §2) are binding on every design decision:

1. **The QR identifies; auth decides.** The QR carries an opaque token, never PHI. Authentication and authorization determine what a doctor sees. (PRD §33: the QR is an access mechanism, not an authentication mechanism.)
2. **Consent is a first-class object** — scoped by grantee, category, duration, and purpose; evaluated on every read.
3. **Tenant isolation by construction.** Tenant context is derived server-side, stamped at the edge, and enforced at gateway, service, and database layers.
4. **Hospitals never write to core stores.** All external data passes through the integration layer with validation, identity resolution, and provenance.
5. **Everything sensitive is audited** in an append-only, tamper-evident store.
6. **API-first.** Every capability is an API; apps and integrations are peers of the same surface (PRD §28).
7. **Stateless services, isolated state.** Compute scales horizontally; isolation lives in the data layer.
8. **Fail loudly, never silently.** Failed clinical data lands in a visible error queue, never dropped.

## 3. Core model: global identity, tenant-owned records, consented composition

This is the central architectural idea (Canvas "Why patients span tenants" panel):

> Patient identity is a **global platform entity**; clinical records are **tenant-owned** with provenance. The patient's consolidated timeline is a consented, read-time composition across tenants — this is what makes cross-hospital history possible without hospitals sharing databases.

Consequences for the architecture:

- The identity registry is a platform-global store with strict service-only access; clinical stores are partitioned per tenant by isolation tier.
- The 360° view is never materialized as a cross-tenant table. It is composed at read time by the records service from each tenant's slice, trimmed by the policy engine's consent evaluation. Consent revocation therefore takes effect immediately — there is no stale composite to purge.
- Every cross-tenant read is a consented composition and is always audited (tenancy invariant 6, §11.4).
- Per-tenant external-ID crosswalks (e.g. hospital MRN `HOSP-928372` → `ATL-82X92K`) live with the identity registry and are the only way integration data attaches to a patient.

### Acceptance criteria

- Deleting or revoking a consent grant removes the corresponding tenant's records from the next 360° composition without any batch job.
- No database table or index contains pre-joined clinical data from more than one tenant.
- Every 360° composition emits audit events attributing the read to actor, patient, tenant(s), and purpose.

## 4. Client tier

Four client types (Canvas layer 1; Architecture doc §4). Full per-screen specs live in [02-patient-app.md](02-patient-app.md), [03-doctor-app.md](03-doctor-app.md), [04-admin-app.md](04-admin-app.md).

| Client | Platform | Responsibilities | Auth |
|---|---|---|---|
| Patient app | iOS / Android | Registration; QR presentation; timeline, records and reports; medications; consent management; family profiles (guardian links); emergency lock-screen card; notifications | OTP, passkeys, Face ID; optional ABHA federation |
| Doctor app | iOS / Android | Patient search (identity-only results); QR scanning; patient 360° with AI clinical brief; clinical writes (diagnosis, prescription, document upload); break-glass emergency access | Org SSO (OIDC/SAML per tenant) + MFA |
| Admin console | Web | Staff and role management; integration configuration and credentials; error queue; publication rules; audit export | Org SSO + MFA |
| Hospital EHR/HIS/LIS | External system | Systems of record; push finalized results via the integration layer only | mTLS + OAuth2 client credentials |

Client-tier requirements:

- The QR screen and the emergency card are cached for offline display; biometric unlock gates cached data.
- Device posture for clinical apps: certificate pinning, jailbreak/root detection, no PHI in push payloads.
- In the LLD component diagram, the Patient App requires `IPatientAPI`, the Doctor App `IClinicalAPI`, the Admin Console `IAdminAPI` — all three provided solely by the API gateway.

### Acceptance criteria

- Patient app renders the QR screen and emergency card with no network connectivity.
- Push notifications received on any client contain no PHI.
- A rooted/jailbroken device is detected and blocked (or restricted) for the doctor app.

## 5. Edge and API tier — tenant resolution happens here

Traffic enters through CDN + WAF (TLS 1.3 termination, DDoS and OWASP filtering, bot filtering, static assets), then the **API gateway**, which is where multi-tenancy begins, then a load balancer (regional routing for data residency, health checks, blue/green deploys).

Gateway responsibilities (Canvas layer 2; Architecture doc §5; LLD "API Gateway" component):

- **Tenant resolver**: resolves the tenant from subdomain (`citygeneral.atlas.health`), mTLS client certificate (system integrations), or token claim, and stamps a signed, immutable **tenant context** — `{tenant_id, tier, region, plan}` — onto the request. Downstream services accept only the signed header; services never trust client-sent tenant ids.
- Per-tenant rate limits and quotas (noisy-neighbor control).
- Stricter anti-replay and nonce checks on QR endpoints.
- Request schema validation; API versioning (`/v1`); idempotency keys on writes; request signing.
- Regional load balancing honors each tenant's data-residency pin.
- Provides the three client-facing interfaces `IPatientAPI`, `IClinicalAPI`, `IAdminAPI` (LLD); it is the only entry point into application services.

### Acceptance criteria

- A request whose body or query string claims a different `tenant_id` than the resolved context is served strictly under the resolved context (client-sent tenant ids are ignored).
- A downstream service rejects any request lacking a validly signed tenant context header.
- Replayed QR-resolve requests (reused nonce) are rejected at the gateway.
- Per-tenant rate limiting demonstrably isolates one tenant's traffic spike from another tenant's latency.

## 6. Identity, authorization, and consent services — "the QR identifies; auth decides"

Security-critical band between the edge and the core services (Canvas layer 3). Four services:

### 6.1 AuthN service

- **Patients**: mobile OTP baseline; passkeys and Face ID once enrolled; optional ABHA (ABDM) federation in India deployments.
- **Doctors and staff**: OIDC/SAML federation to the tenant's IdP, MFA enforced, professional-registry verification hook (HPR-ready). Short-lived sessions with refresh; device binding for clinical apps.
- **System clients** (hospital integrations): mTLS + OAuth2 client-credentials, scoped per tenant; secrets held in the platform secrets manager, never exposed to users.

Detail: [06-identity-auth.md](06-identity-auth.md).

### 6.2 QR token service

- Issues opaque, short-lived rotating tokens — no identifiers or PHI in the QR. LLD interface `IQRToken`: `issue · resolve · revoke · rotate`, token TTL 5 minutes (`{ttl: 300s, rotating}` in the QR-consult sequence).
- Resolution binds the token to the scanning doctor's session (session binding), consumes a nonce (anti-replay), and is rate-limited per doctor and per tenant.
- Token state lives in a replicated cache (Redis-class) with strict TTL; issuing and resolution are both audited.
- Printed/static fallback QRs (patients without phones) resolve to a higher-friction flow requiring an additional identity check.

Detail: [08-qr-subsystem.md](08-qr-subsystem.md).

### 6.3 Policy engine (AuthZ)

- Evaluates every access: **RBAC** (patient, doctor, org-admin, platform roles) layered with **ABAC** (tenant, facility, specialty, purpose-of-use) and **consent grants**. LLD interface `IAuthz`.
- Consent grant = `{patient, grantee (doctor/org), categories, duration, purpose, status}`. Categories: general history, reports & labs, medications, diagnoses, mental/behavioral health (default off), sensitive clinical data.
- Grants are created by QR scan (implicit consult grant with default duration — 30 days in the LLD sequence), by explicit patient approval of a request (24 h / 30 d / 90 d durations), or by organization policy for records the org itself created. All grants auto-expire and are revocable.
- **Break-glass**: emergency access with mandatory reason, limited to the emergency profile plus critical history, immediately alerting compliance and the patient, flagged in audit for review.
- Policy decisions are cached seconds-scale and logged with their inputs, for explainability.

Detail: [07-consent-and-access-control.md](07-consent-and-access-control.md).

### 6.4 Tenant directory

- Registry of tenants: tier, region, IdP configuration, feature flags, branding.
- Org and facility hierarchy; practitioner registry (HPR-ready).

### Acceptance criteria

- No QR payload decodes to any patient identifier or PHI; tokens are resolvable only by the QR token service.
- A resolved QR token cannot be reused from a different doctor session (session binding) or replayed (nonce).
- Every policy decision is reproducible from its decision log entry (inputs + outcome).
- Revoking a grant causes the next read to be denied within the policy-cache TTL (seconds).

## 7. Core domain services

Eight stateless, tenant-aware, horizontally scalable services (Canvas layer 4; Architecture doc §9). Every read/write carries `tenant_id + actor + purpose`. LLD groups these in the «package» APPLICATION SERVICES with provided interfaces noted.

| # | Service | LLD interface | Responsibilities | Key data |
|---|---|---|---|---|
| 1 | Patient identity & registry | — (service-only) | Global Atlas Patient ID (cross-tenant); demographic golden record; per-tenant external-ID crosswalk; probabilistic matching with confidence thresholds — ambiguous matches route to human resolution, never auto-attach; family/guardian links | Identity registry DB |
| 2 | Medical records | `IRecords` | Timeline events, diagnoses, prescriptions, hospitalizations, discharge summaries; FHIR-aligned internal model; record versioning; report lifecycle ORDERED → IN_PROGRESS → COMPLETED → VERIFIED → FINAL → PUBLISHED_TO_PATIENT; provenance retained on every synchronized record; 360° composition | Clinical DB |
| 3 | Consent & access | `IConsent` | Grant lifecycle (request, approve, deny, revoke, expire); grants `{grantee, categories, duration, purpose}`; patient-visible access log; emergency-access records | Clinical DB |
| 4 | Documents | — | Upload/download via pre-signed URLs; malware-scanning pipeline before records become visible; metadata and linkage to timeline events; PDF, images, DICOM | Object storage |
| 5 | AI summary | `ISummary` | Patient-friendly summaries, clinical briefs at scan time, ask-the-record Q&A; generated only from the requester's *authorized slice*; citations to source records; disclaimers; per-tenant configuration; no cross-tenant training; prompts and outputs logged | Stateless over records API + vector store |
| 6 | Search | — | Patient lookup by name, ID, phone, DOB; results expose identity only until an authorization check passes; per-tenant index partitions rebuilt from source of truth | Search index |
| 7 | Notifications | — | Push/SMS/email; PHI-free templates; consent requests, report-ready, access alerts, follow-up reminders; per-tenant and per-user preferences | Queue + prefs |
| 8 | Audit | `IAudit` | Append-only, hash-chained events for every view, write, grant, QR scan, break-glass, export; `{actor, role, tenant, patient, action, resource, method, result, session}`; separate credentials and WORM retention; per-tenant export | Audit store |

### 7.1 AI summary service — architecture summary

Product behavior is specified in [05-ai-features.md](05-ai-features.md); the architectural contract (Architecture doc §10) is:

- **Capabilities**: patient summary (regenerated when new records publish), clinical brief at QR scan, ask-the-record Q&A, per-report explanation. Deferred post-MVP: drug-interaction detection, coding suggestions, clinical decision support — the pipeline hosts them without rework.
- **RAG pipeline**: `authorize → retrieve → assemble → generate → verify → respond`. Grounding is computed per request from the requester's authorized slice, never from a pre-built cross-record corpus — so consent revocation takes effect instantly. Retrieval is hybrid: structured pulls from the FHIR-aligned records API (active problems, meds, latest labs always included) plus semantic search over embedded record chunks. Documents and notes are injected as quoted data, never as instructions (prompt-injection boundary). Verification checks every factual claim (numbers, dates, drug names) against retrieved facts; unsupported claims are dropped or the response refused; output filters catch PHI leakage outside the slice. Responses carry inline citations and fixed disclaimers ("not medical advice" / "verify before clinical decisions"); prompt, retrieved set, and output are logged to audit.
- **Provider gateway**: one internal interface (`complete`, `embed`), N adapters — Azure OpenAI, AWS Bedrock (Claude), GCP Vertex, self-hosted vLLM for no-egress cells. No service talks to a vendor SDK directly. Provider, model, and region are per-tenant configuration; BAA/DPA, zero retention, and no-training are hard contract requirements. Per-use-case model routing — a small/fast model for patient summaries, a stronger model for clinical briefs and Q&A. Failover chains per use case, timeouts, circuit breakers, model version pinning with staged upgrades, per-tenant token metering feeding control-plane billing.
- **Embeddings**: generated at ingest (re-embedded on amendment) into a per-tenant encrypted vector store (pgvector / OpenSearch k-NN), scoped by patient, disposable and rebuilt from source of truth. Vectors are treated as PHI. Summary cache keyed by `{patient, authorized-slice hash, model version}`. Scan-time brief SLO: p95 < 2.5 s (precomputed on `report.finalized` where the tenant enables it).
- **Safety**: AI output is never written to the record and triggers no clinical action; clinician-reviewed golden sets gate model/prompt changes; red-teaming for prompt injection via uploaded documents; every generation logged (requester, slice hash, model, prompt, output).

### Acceptance criteria

- Each of the eight services deploys and scales independently; none holds request state between calls.
- Search results for a doctor without consent contain identity fields only — no clinical data.
- A document is not visible in any timeline until its malware scan status is clean.
- An AI response contains no fact absent from its retrieved authorized slice, and every response carries citations and the fixed disclaimers.
- Audit events are hash-chained; altering any historical event is detectable.

## 8. Eventing and async processing

An event bus (Kafka-class) decouples services (Canvas event-bus band; Architecture doc §12):

```text
Topics: report.finalized · record.created · consent.granted · consent.revoked
        · access.viewed · integration.failed · notification.requested
```

- Producers use the **transactional outbox pattern** (LLD: Records service → outbox → Event Bus); consumers are idempotent.
- Partitions are keyed by `tenant_id`, giving per-tenant ordering where required.
- Async consumers: notification dispatch, search indexing, audit projection, AI pre-computation (where the tenant enables it), webhook fan-out to tenant systems.

### Acceptance criteria

- A service crash between DB commit and event publish loses no events (outbox replay).
- Redelivered events produce no duplicate side effects (idempotent consumers).
- Events for one tenant are never observable by another tenant's consumers or webhooks.

## 9. Integration layer — the only door for external clinical data

Hospitals never write to core stores; per-tenant connectors carry their own credentials, identifier mappings, resource whitelists, and publication rules. One logical pipeline; adapters plug in without touching the core model. Full detail: [11-integrations.md](11-integrations.md).

- **Protocol adapters**: REST and FHIR R4 first; HL7 v2 messaging; webhooks/event push; secure file transfer (SFTP) for legacy systems; scheduled sync where events are unavailable.
- **Processing pipeline**:

```text
authenticate source → validate schema → resolve patient identity → map/normalize
  → dedupe → persist with provenance → apply publication policy → emit events → notify
```

- Idempotency key: `{tenant, source_system, source_record_id, version}` — duplicates are detected and acknowledged (source gets 200; safe replay) without reprocessing.
- Processing states, all visible to tenant admins with PHI-minimized failure reasons:

```text
RECEIVED → VALIDATING → PROCESSED | REJECTED | RETRYING → FAILED → DEAD_LETTER
```

- Error handling: retry with exponential backoff; per-tenant dead-letter queues surfaced in the admin error queue UI; ambiguous identity matches route to the human resolution queue — never auto-attach to the wrong patient; nothing silently dropped. Failed source authentication returns 401, alerts the tenant admin, and is audited (LLD activity diagram).
- Provenance preserved verbatim: source system, source organization, source record ID, original clinical timestamp, author, record status, import time, version. Transformations never overwrite provenance. Amendments create new versions; published records visibly indicate when an updated version exists.
- Publication policy per tenant/report type — one of the four PRD §24 modes: auto-publish after finalization, publish after verification, hold for manual release, or restricted to healthcare professionals (clinician-only).

### Acceptance criteria

- Replaying the same source payload twice yields exactly one persisted record and a success acknowledgment both times.
- A schema-invalid payload lands in DEAD_LETTER with a PHI-minimized reason visible in the admin error queue.
- An identity match below the confidence threshold is never attached automatically; it appears in the resolution queue.
- Every synchronized record can answer: which system produced it, under which source ID, when, by whom, at which version.

## 10. Data layer

Isolation by tier, encrypted per tenant (Canvas layer 6; Architecture doc §13). All persistence goes through a tenant-aware data access layer — the repository enforces `tenant_id`; no raw cross-tenant queries. Entity detail: [09-data-model.md](09-data-model.md).

| Store | Technology class | Isolation | Notes |
|---|---|---|---|
| Clinical DB | PostgreSQL | RLS / schema / cluster by tier | Partitioned by tenant; read replicas for patient-360 composition |
| Identity registry | PostgreSQL | Platform-global, service-only access | Golden records + per-tenant crosswalk |
| Documents | Object storage (S3-class) | Per-tenant bucket/prefix | Pre-signed URLs, lifecycle rules, legal hold |
| Search | OpenSearch-class | Per-tenant partitions/aliases | Disposable — rebuilt from source of truth; no PHI beyond indexed fields |
| Audit | WORM store | Per-tenant streams | Hash-chained, anchored daily, immutable retention, separate credentials from app DBs |
| Cache / sessions / QR state | Redis-class | Tenant-namespaced keys | TTL-bound; no durable PHI |
| Event bus | Kafka-class | Tenant-keyed partitions | Outbox-fed; DLQs per tenant |
| Vector store | pgvector / OpenSearch k-NN | Per-tenant, per-patient scope | Embeddings are PHI; disposable, rebuilt from source |

Cross-cutting data policies:

- **Encryption**: AES-256 at rest everywhere; envelope encryption with per-tenant data keys wrapped by KMS master keys; key rotation; optional BYOK for the dedicated tier.
- **Residency**: each tenant is pinned to a regional cell; backups and replicas stay in-region.
- **Retention & deletion**: policy-driven per record class and jurisdiction; secure deletion honoring audit-retention obligations.

### Acceptance criteria

- Disabling the repository layer's tenant filter in a test still blocks cross-tenant rows (RLS backstop).
- Compromise of one tenant's data key exposes no other tenant's data.
- Search and vector stores can be dropped and fully rebuilt from the clinical DB with no data loss.
- Audit-store credentials do not grant access to any application database, and vice versa.

## 11. Multi-tenancy model

### 11.1 Control plane vs. data plane

A shared **control plane** (no PHI) manages tenant lifecycle; the **data plane** serves clinical traffic and is partitioned per tenant according to tier. Control-plane functions (Canvas panel):

- **Tenant provisioning** — create tenant → pick tier & region → provision schema/DB, storage prefix, KMS key, IdP config, subdomain, quotas; automated and idempotent.
- **Plans & metering** — per-tenant usage (API calls, storage, seats), feature flags, billing hooks (including AI token metering, §7.1).
- **Config & branding** — publication rules, notification preferences, report-type policies, logo/locale per tenant.
- **Lifecycle** — suspend, export (patient-portable data out), offboard with secure deletion and retention holds.

### 11.2 Isolation tiers — same code, different isolation

| Tier | Compute | Database | Storage / search | When to use |
|---|---|---|---|---|
| **Pooled** | Shared | Shared tables + `tenant_id` + Postgres row-level security | Per-tenant prefixes / index aliases | Clinics, small labs — starter plan; cheapest, fastest onboarding; per-tenant encryption keys still apply |
| **Siloed** | Shared, optional dedicated queue consumers | Schema-per-tenant on shared clusters | Dedicated bucket + index partition | Hospitals — standard plan; balances cost and blast-radius |
| **Dedicated** | Dedicated cell | Own DB cluster (or full cell), pinned region, optional BYOK | Fully dedicated | Hospital networks, government — enterprise plan; custom SLOs; required for national-platform deployments |

### 11.3 Per-tenant envelope encryption

Every tier — including pooled — uses envelope encryption: per-tenant data keys wrapped by KMS master keys, with rotation. A leaked key exposes one tenant only. Dedicated tier can bring its own keys (BYOK).

### 11.4 Tenancy invariants

Verbatim from the Canvas "Multi-tenancy invariants" panel; these are non-negotiable and testable:

> 1 · tenant_id derived server-side, never from client input
> 2 · Every query passes through the tenant-aware repository; RLS is the backstop
> 3 · Per-tenant encryption keys — a leaked key exposes one tenant only
> 4 · Quotas & rate limits per tenant — no noisy neighbors
> 5 · Events, indexes, caches, DLQs all namespaced by tenant
> 6 · Cross-tenant reads exist ONLY via patient-consented composition, always audited
> 7 · Offboarding exports then provably deletes — audit retained per law

### Acceptance criteria

- Provisioning a new pooled tenant is fully automated, idempotent, and completes in minutes.
- Each invariant above has at least one automated test or audit control asserting it in CI/production.
- Moving a tenant between tiers requires no application code change (configuration and data migration only). *Proposed:* tier migration runbook is part of control-plane documentation.
- Offboarding produces a portable export, then provably deletes tenant data while retaining audit records per regulation.

## 12. Security architecture — defense in depth

Five enforcement layers (Architecture doc §14; Canvas security panel):

1. **Edge**: WAF, DDoS protection, TLS 1.3, certificate pinning in apps.
2. **Gateway**: tenant resolution, signed context, rate limits, schema validation, anti-replay.
3. **Service**: authN + policy engine on every call; purpose-of-use recorded; input validation; secure file handling with malware scanning.
4. **Data**: tenant-aware repositories, RLS backstop, per-tenant keys, least-privilege DB roles, secrets manager (integration credentials never exposed).
5. **Operations**: SAST/DAST in CI, dependency scanning, pen tests, admin access via JIT elevation with audit, no standing production access to PHI.

Compliance posture: DPDP (India), HIPAA, GDPR served via regional cells; data residency pinned per tenant region. India-first alignment with ABDM/ABHA (see [06-identity-auth.md](06-identity-auth.md)).

## 13. API surface

Versioned REST (`/v1`), FHIR-compatible read endpoints planned. Full contract in [12-api-and-interoperability.md](12-api-and-interoperability.md); the domain list (PRD §28 + Architecture doc §15):

```text
/auth /patients /doctors /organizations /encounters /diagnoses /prescriptions
/medications /reports /documents /hospitalizations /discharge-summaries
/consents /access /audit /qr /notifications /integration
```

`/integration` is tenant-scoped ingest. Webhooks deliver tenant-facing events with signed payloads. The API layer is designed for future integration with hospitals, diagnostic laboratories, pharmacies, insurance providers, healthcare applications, national/regional health platforms, and Electronic Health Record systems.

## 14. Key end-to-end flows

Five flows define correct system behavior. Sequence-diagram detail (participants, message order) is in the LLD "Sequence" tabs.

### 14.1 Flow A — QR consult (synchronous)

1. Patient opens the QR screen; app fetches a fresh rotating token (`POST /qr/tokens` → token `{ttl: 300s, rotating}`, issued with tenant context).
2. Doctor scans; gateway resolves the doctor's tenant, checks rate limits and the anti-replay nonce (`POST /qr/resolve {token, nonce}`).
3. QR token service resolves token → Atlas Patient ID and binds it to the doctor's session.
4. Policy engine evaluates `(actor, patient, purpose)` — role + consent; on PERMIT, a consult grant is created with default duration (30 d in the LLD sequence) and categories.
5. Records service composes the authorized 360° slice (cross-tenant, consent-trimmed); the AI service generates the clinical brief from that slice only, with verified claims and citations.
6. Doctor app receives the 360° view + AI brief. Audit events written asynchronously (scan, grant, view); patient notified "Dr. X viewed your records."

SLO: QR-to-patient p95 < 500 ms; patient-360 load < 1 s; AI brief p95 < 2.5 s (§15).

### 14.2 Flow B — Lab report sync (asynchronous)

1. LIS finalizes a report and pushes it via the tenant's connector (`POST /integration/fhir/Bundle` over mTLS; FHIR/HL7/file). Gateway acknowledges `202 Accepted {event_id}` and enqueues with the idempotency key.
2. Pipeline authenticates the source, validates schema, resolves the patient (crosswalk `HOSP-928372` → `ATL-82X92K`, confidence-scored; ambiguity → admin resolution queue).
3. Record persisted with full provenance (stored v1, status FINAL); publication policy applied (one of the four PRD §24 modes: auto-publish after finalization, publish after verification, hold for manual release, or restricted to healthcare professionals).
4. `report.finalized` event → patient notification ("New report available", PHI-free push), search indexing, audit.
5. Failures land in RETRYING or the per-tenant DLQ, visible in the admin error queue.

### 14.3 Flow C — Consent request

1. Doctor (or org) requests access; patient gets a PHI-free notification.
2. Patient reviews the requester, picks duration (24 h / 30 d / 90 d) and categories (sensitive categories default off), approves or denies.
3. Grant stored; policy cache invalidated; both parties notified; audit written; grant auto-expires.

### 14.4 Flow D — Break-glass emergency access

1. Doctor selects emergency access with a mandatory reason and patient identifier.
2. Policy engine issues a constrained emergency grant (emergency profile + critical history only).
3. Compliance and the patient are alerted immediately; the session is flagged in audit for post-hoc review.

### 14.5 Flow E — Tenant onboarding

1. Control plane creates the tenant: tier, region, subdomain, quotas.
2. Provisioning automation creates schema/DB, storage prefix, KMS key, search partition, IdP config — idempotent, minutes not days.
3. Admin configures integrations (credentials, mappings, publication rules) and invites staff; test events flow through a sandbox connector before go-live.

### Acceptance criteria

- Flow A completes with all six steps auditable from a single trace ID; the patient's access log shows the scan.
- Flow B delivered twice by the LIS results in one record and two success acknowledgments.
- Flow C denial leaves no grant and no access; approval is effective on the next read.
- Flow D access without a reason string is impossible; every break-glass session appears in a compliance review queue.
- Flow E ends with a tenant that can pass a sandbox integration test before any production data flows.

## 15. Deployment and operations

- **Topology**: regional cells (e.g., India, EU, US), each multi-AZ. Pooled and siloed tenants share cell infrastructure; dedicated tenants get their own cell or cluster. Residency never crosses cells.
- **Runtime**: containerized services on Kubernetes-class orchestration; horizontal autoscaling per service — API, records, integration workers, and AI scale independently (PRD §31 scalability: API, database, document storage, authentication, search, audit all scale independently).
- **Delivery**: trunk-based CI/CD; blue/green or canary deploys; schema migrations gated and reversible; per-tenant feature flags for progressive rollout.
- **Observability**: structured logs, metrics, traces — all tagged `tenant_id`; OpenTelemetry. Per-tenant dashboards and SLOs; noisy-neighbor detection tied to quotas; integration lag and DLQ depth alerts surfaced to tenant admins.

Observability panels (minimum set, from Canvas/Architecture observability items):

| Panel | Key signals | Alert |
|---|---|---|
| QR access | QR-to-patient resolve latency (p95 target < 500 ms), scan volume, nonce-replay rejections | p95 breach; replay spike |
| Patient 360° | Composition load time (target < 1 s), read-replica lag | Load-time breach |
| AI | Brief latency (p95 < 2.5 s), verification-drop rate, provider failovers, per-tenant token spend | Latency/failover breach |
| Integration | Per-tenant lag, pipeline state counts, DLQ depth | Lag threshold; DLQ depth — alert tenant admins |
| Tenancy | Per-tenant quota consumption, rate-limit hits | Noisy-neighbor detection |
| Audit | Append rate, hash-chain anchor status | Chain-anchor failure |

*Proposed:* the panel grouping above is an implementation arrangement of the SLOs and alerts named in the sources; the individual signals and thresholds are all sourced.

## 16. Reliability and disaster recovery

- Multi-AZ by default; regional cells give in-region failover that preserves residency.
- **RPO ≤ 5 min, RTO ≤ 1 h for clinical reads.**
- Point-in-time-recovery backups per tenant DB/schema; regular restore drills; integrity checks on clinical records; durable records with safe migration mechanisms (PRD §31 reliability).
- Read replicas absorb heavy patient-360 queries.
- Graceful degradation: QR presentation and the emergency card serve from cache if core services degrade; reads outlive writes.

### Acceptance criteria

- A restore drill recovers a single tenant's schema to a point in time without touching other tenants.
- With the clinical DB write path down, QR presentation and emergency-card reads still succeed.
- Failover within a region completes inside RTO with data loss inside RPO, verified by game-day exercise.

## 17. Suggested technology stack

As proposed in Architecture doc §19 — suggestions, not mandates; the architecture constrains classes of technology, not vendors:

| Layer | Suggestion |
|---|---|
| Mobile apps | Native iOS/Android or React Native (shared design system) |
| Edge / gateway | CloudFront/Cloudflare + WAF; Kong/Envoy-based gateway |
| Services | Kotlin/Java or Go microservices; gRPC internal, REST external |
| Policy engine | OPA/Cedar-class with consent extension |
| Data | PostgreSQL (RLS), Redis, Kafka, OpenSearch, S3-class storage, KMS/HSM |
| AI | AI provider gateway (Azure OpenAI / Bedrock / Vertex / self-hosted vLLM adapters); RAG over the authorized slice; pgvector or OpenSearch k-NN for embeddings |
| Infra | Kubernetes, Terraform, per-cell isolation; OpenTelemetry observability |

## 18. MVP boundary and evolution

**In MVP** (Architecture doc §20; PRD §30): patient/doctor/org registration; QR generate + scan + authorized lookup; patient overview and timeline; diagnoses; prescriptions; document upload/view; discharge summaries; basic consent; audit logging; one integration path (REST/FHIR ingest of finalized reports); identity mapping; patient notifications; pooled + siloed tiers in one region. AI summaries are in scope as a top-priority addition (patient summary, scan-time clinical brief, ask-the-record — see [05-ai-features.md](05-ai-features.md)); the MVP explicitly defers AI *decision support* (§7.1).

**Deferred**: full FHIR interoperability surface, HL7 v2 breadth, insurance/pharmacy integrations, dedicated cells and BYOK, advanced analytics, clinical decision support, complex healthcare billing, multi-country compliance abstractions (PRD §30).

**Evolution without re-architecture**: the FHIR-aligned model, provenance capture, and the integration boundary are designed so none of the deferred items require re-architecture. The platform grows toward a portable health identity and interoperability layer — cross-hospital history, consent marketplace, national platform federation — without touching the core (PRD §34).

### Acceptance criteria

- MVP ships with the pooled and siloed tiers only, in one region, and adding the dedicated tier later changes configuration and infrastructure, not service code.
- Adding a second integration protocol (e.g. HL7 v2) adds an adapter without modifying the processing pipeline or core model.
- Turning on a deferred AI capability reuses the existing RAG pipeline stages unchanged.
