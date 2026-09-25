# 11 — Hospital & EHR Integration

How external hospital systems (EHR/HIS/LIS/PACS) publish clinical data into Atlas: the integration boundary, supported methods and per-method contracts, per-organization configuration, patient identity resolution, the ingest pipeline, report lifecycle and publication, eventing, error handling, and provenance.

Sources: `Atlas_Project_Requirements.md` §§19–27 (plus §3.2 and §6 for manual document upload); `Atlas Architecture.dc.html` §4 (admin console), §6 (system-client auth), §11 (integration layer), §12 (eventing), §16 flows B and E, §20 (MVP boundary); `Atlas LLD.dc.html` — "Sequence · Report sync" diagram, "Activity · Ingest" flowchart, domain class model (IdentityMapping, TimelineEvent, Report, TenantOrganization); `Canvas.dc.html` lane 5 (integration layer); `Atlas App.dc.html` — Admin dashboard, Admin integration queue, patient Report detail (provenance card), Records list (status pills).

Sibling specs: data entities in [09-data-model.md](09-data-model.md); overall service topology in [10-architecture.md](10-architecture.md); API surface and FHIR alignment in [12-api-and-interoperability.md](12-api-and-interoperability.md); the admin screens that surface integration state in [04-admin-app.md](04-admin-app.md); notification behavior in [02-patient-app.md](02-patient-app.md).

## 1. Integration boundary and primary flow

### 1.1 Boundary rule

Atlas is **not** the hospital's HIS. Hospitals keep their existing EHR/HIS/LIS as the system of record for clinical operations; Atlas is the patient-centric aggregation and access layer on top (PRD §19). Integration is **one-way clinical publication**: finalized clinical information flows from the hospital system into Atlas; Atlas never writes back into hospital systems and never becomes an operational ordering/workflow system for the hospital.

Hard invariant (PRD §19, Canvas lane 5, Architecture principle): **hospitals never write directly to the Atlas core database.** All external clinical data passes through the integration layer, which is the only door for external clinical data. The mandatory chain is:

```text
Hospital System
      ↓
Integration API / Gateway
      ↓
Authentication & Validation
      ↓
Patient Identity Resolution
      ↓
Data Mapping / Normalization
      ↓
Clinical Record Processing
      ↓
Atlas Medical Record
      ↓
Notification / Access / Audit
```

This boundary keeps Atlas's core medical-record model stable while any number of hospitals integrate independently, each through its own per-tenant connector with its own credentials, identifier mappings, resource whitelists, and publication rules (Architecture §11).

### 1.2 Primary integration flow

```text
Hospital EHR / HIS / LIS
          │
          ▼
 Atlas Integration Layer
          │
          ├── Identity Resolution
          ├── Validation
          ├── Data Mapping / Normalization
          └── Event Processing
          │
          ▼
     Atlas Records
          │
       ┌──┴──┐
       ▼     ▼
    Patient Doctor
```

### 1.3 Worked example — laboratory report (PRD §19)

1. A doctor orders a laboratory test in the hospital's existing EHR. Atlas is not involved.
2. The order is processed by the hospital/LIS.
3. The laboratory completes the test.
4. The LIS/EHR marks the report as verified/finalized.
5. The hospital system sends the finalized result to Atlas via the tenant's configured connector.
6. Atlas validates the source (mTLS + OAuth2 client-credentials) and resolves the patient identity (`HOSP-928372` → `ATL-82X92K`).
7. Atlas stores the report and associates it with the patient timeline, with full provenance.
8. Atlas publishes the report according to the tenant's publication rules and the patient's access/visibility policy.
9. The patient (Priya Sharma) receives a notification that a new report is available.
10. The patient views the report in the Atlas app (Report detail screen, with provenance card).
11. Authorized doctors see the report as part of the patient's consented 360° history.

The same model carries finalized diagnoses, prescriptions, imaging reports, discharge summaries, procedures, and the other resources in §5.1.

### 1.4 Demo integrations (fixed by the prototype)

The prototype's admin dashboard for City General Hospital shows three integrations; specs and demos use these consistently:

| Integration | Method | Dashboard copy | Health state shown |
|---|---|---|---|
| Apollo LIS | FHIR | "FHIR · 1,204 events today · healthy" | Healthy (green dot) |
| Radiology PACS | Webhooks | "Webhooks · 96 events today · healthy" | Healthy (green dot) |
| Legacy HIS | File-based | "File-based · last sync 08:40" | Degraded (red dot, "3 failed ›" pill, red border) |

Acceptance criteria:
- No API path exists by which an external system writes to the clinical, identity, or audit stores without passing through the integration pipeline.
- An integrated hospital can continue ordering and finalizing in its own EHR with zero Atlas-side workflow steps; only finalized publication touches Atlas.
- The end-to-end lab flow in §1.3 completes with the report visible on the patient timeline, a patient notification sent, and provenance retained.

## 2. Integration methods (PRD §20)

Hospitals have very different stacks, so the layer supports multiple mechanisms behind protocol adapters that all normalize into **one internal pipeline** — new protocols never touch the core medical-record model (Architecture §11.1).

| # | Method | Priority | Transport | Typical source | Demo instance |
|---|---|---|---|---|---|
| 1 | REST API ingest | MVP | HTTPS + mTLS, OAuth2 client-credentials | Modern HIS | — |
| 2 | HL7 FHIR R4 API | MVP | HTTPS + mTLS; FHIR Bundles | LIS/EHR with FHIR support | Apollo LIS |
| 3 | Webhooks / event push | MVP | HTTPS callbacks into Atlas ingest endpoint | Event-capable systems | Radiology PACS |
| 4 | HL7 v2 messaging | Post-MVP adapter | MLLP/secure channel via adapter | Older EHR/LIS | — |
| 5 | Secure file-based batch | Supported for legacy | SFTP / file drop, scheduled pickup | Legacy HIS | Legacy HIS ("last sync 08:40") |
| 6 | Scheduled synchronization | Supported | Polling where events are unavailable | Any | — |
| 7 | Manual upload | Supported | In-app (doctor "Add to record" flow, PRD §3.2; stored in the document store, PRD §6) | Humans | Doctor "Add to record" |

MVP prioritizes REST APIs, FHIR-compatible interfaces, and webhooks (PRD §20); the MVP ships **one** integration path — REST/FHIR ingest of finalized reports — plus identity mapping and patient notifications (Architecture §20).

### 2.1 FHIR API contract

- Endpoint: `POST /integration/fhir/Bundle` under the tenant-scoped `/integration` API domain (LLD report-sync sequence; Architecture §15).
- Transport auth: mTLS client certificate + OAuth2 client-credentials, scoped per tenant; the gateway resolves the tenant from the client certificate and stamps the signed tenant context (Architecture §6, §5).
- Synchronous response: `202 Accepted` with an `event_id` the source can use to correlate status. Processing is asynchronous from this point.
- Payload: FHIR R4 Bundle containing declared resource types from the tenant's resource whitelist (§3). Schema validation is stage 2 of the pipeline (§5.2).
- Idempotency: derived key `{tenant, source_system, source_record_id, version}`; redelivery of the same key is acknowledged without reprocessing (§7.2).

```text
POST /integration/fhir/Bundle          (mTLS + OAuth2 client-credentials)
→ 202 Accepted { event_id }
→ async: enqueue with idempotency key → pipeline (§5)
```

### 2.2 Webhooks / event push contract

- The source system pushes an event notification when clinical information changes (e.g., PACS finalizes an imaging report); the payload either carries the resource or a reference the adapter fetches. Proposed: reference-style payloads fetch via the tenant connector's configured API endpoint and credentials, so the webhook itself never needs to carry PHI.
- Same tenant authentication, `202 Accepted {event_id}` acknowledgement, idempotency, and pipeline as §2.1.
- Distinct from **outbound** tenant-facing webhooks (Atlas → hospital), which deliver signed payloads for events the tenant subscribes to (Architecture §15, §12); those are specified in [12-api-and-interoperability.md](12-api-and-interoperability.md).

### 2.3 File-based batch contract (legacy)

- Secure file transfer (SFTP/file drop) with scheduled pickup; the admin dashboard shows the last sync time ("last sync 08:40").
- Each file decomposes into individual events; every event gets its own idempotency key and independently traverses the pipeline, so one bad record fails alone rather than poisoning the batch. (Decomposition granularity is Proposed; the prototype's error queue shows per-record failures from the file-based Legacy HIS, which implies it.)
- Batch-source failures are the demo content of the admin error queue (§8.4): a lab result, a discharge summary, and an imaging report failing independently.

### 2.4 Manual upload

- Doctors add consultation notes, diagnoses, prescriptions, and uploaded reports/photos through the doctor app's Add to record flow (PRD §3.2: "Upload medical documents/reports"; flow spec in [03-doctor-app.md](03-doctor-app.md)); the document store itself is PRD §6. These are Atlas-originated records, not synchronized ones, and Atlas distinguishes the two (PRD §23) — but manually uploaded *external* documents still carry provenance metadata (uploader, upload time, stated source). Proposed: manual uploads bypass source authentication and schema validation (the app session covers both) but share persistence, publication, eventing, and audit with the pipeline from the "persist + provenance" stage onward.

Acceptance criteria:
- A FHIR Bundle POST with a valid tenant client certificate returns `202 Accepted` with an `event_id` in under the gateway's synchronous budget; the record then appears via the async pipeline.
- The same Bundle re-sent (same `{tenant, source_system, source_record_id, version}`) returns success and creates no duplicate record.
- A request with a missing/invalid client certificate or OAuth token is rejected `401` and never enqueued.
- Adding a new protocol adapter requires no change to the internal record model or pipeline stages.

## 3. Hospital integration configuration (PRD §21)

Integration is configured per organization by a tenant admin in the admin console ("integration configuration and credentials, error queue, publication rules" — Architecture §4; UI in [04-admin-app.md](04-admin-app.md)). Connector configuration lives in the control plane; the connector executes in the data plane with the tenant's credentials.

| Configuration item | Content |
|---|---|
| Organization identity | Tenant/organization the connector belongs to (`tenant_id`) |
| Integration credentials | mTLS client certificate, OAuth2 client id/secret — held in the platform secrets manager, **never exposed to ordinary users** (PRD §21; Canvas security lane: "integration creds never exposed") |
| API endpoints | Source-system endpoints (for fetch-style webhooks / scheduled sync) and the Atlas ingest endpoint issued to the source |
| Protocol | Which adapter: REST, FHIR R4, HL7 v2, webhook, file-based, scheduled sync |
| Patient identifier mapping | Which external identifier scheme maps to the crosswalk (§4) |
| Supported clinical resources | Resource whitelist — only declared types are accepted (§5.1) |
| Report publication rules | Per report type: auto-publish / publish after verification / hold for manual release / clinician-only (§6.2); stored on `TenantOrganization.publication_rules` (LLD class model) |
| Notification preferences | Whether/how patients are notified on publication |
| Synchronization mode | Event-driven, scheduled, or batch |
| Webhook configuration | Inbound push settings and outbound tenant-facing webhook subscriptions |
| Integration status | Enabled / disabled; health shown on the admin dashboard (§1.4) |
| Error/retry configuration | Retry policy parameters, DLQ visibility (§8) |

Lifecycle: admin configures credentials, mappings, and publication rules, then runs **test events through a sandbox connector before go-live** (Architecture §16 flow E — tenant onboarding). Disabling a connector stops ingestion for that source without touching already-persisted records. Proposed: disabling mid-flight lets in-progress events finish their current pipeline pass, then pauses RETRYING events in place rather than failing them.

Acceptance criteria:
- Integration secrets are retrievable only by the connector runtime via the secrets manager; no admin-console API returns secret material after initial entry.
- An event carrying a resource type outside the tenant's whitelist fails schema validation into a visible state (`DEAD_LETTER`, §5.2 stage 2) with a stated reason, not silently ignored.
- A disabled connector accepts no new events (`401`/`403` at the gateway) and its dashboard card reflects the disabled state.
- Sandbox test events are clearly segregated from production clinical data.

## 4. Patient identity resolution (PRD §22)

### 4.1 Model

A hospital has its own patient identifier; Atlas has the global Atlas Patient ID. Atlas maintains a secure per-tenant crosswalk between them in the platform-global identity registry (service-only access — Architecture §13):

```text
Hospital Patient ID: HOSP-928372
                    │
                    ▼
             Identity Resolution      (crosswalk + probabilistic matching)
                    │
                    ▼
Atlas Patient ID: ATL-82X92K          (confidence 0.98 in the demo sequence)
```

`IdentityMapping` entity (LLD class model; full schema in [09-data-model.md](09-data-model.md)):

```text
IdentityMapping {
  tenant_id:    FK
  external_id:  string        // e.g. "HOSP-928372"
  atlas_id:     FK            // e.g. ATL-82X92K
  confidence:   float
  status:       matched | pending
}
```

### 4.2 Match strategy

1. **Crosswalk hit** — the `{tenant_id, external_id}` pair already has a `matched` mapping: resolve directly.
2. **Probabilistic match** — no mapping exists: match demographic attributes against the golden record with confidence scoring and thresholds (Architecture §9: "probabilistic matching with confidence thresholds"). PRD §22 requires "appropriate matching attributes and confidence checks"; Proposed concrete attribute set (golden-record fields in the class model — name, dob, sex, ABHA address — plus phone, a patient-lookup attribute per Architecture §9): name, date of birth, sex, phone, and ABHA address where present, with exact-identifier matches (ABHA) weighted above demographic similarity.
3. **Confident match** — score at or above the auto-link threshold: create the mapping, continue the pipeline. The demo sequence resolves at confidence 0.98.
4. **Ambiguous** — multiple candidates or score in the gray zone: route the event to the **human resolution queue**. The system must prevent duplicate Atlas patient records where an existing patient can be confidently identified (PRD §22).

### 4.3 Never auto-attach

Hard constraint (PRD §22; LLD activity diagram `«constraint» never auto-attach to wrong patient`; Canvas: "ambiguous identity → human resolution, never auto-attach"): an ambiguous match is **never** attached automatically. It parks in the human resolution queue, surfaced in the admin error queue with the exact prototype presentation:

- Card title: `Lab result LAB-102993`, status pill `FAILED` (red).
- Reason line: `HOSP-928372 → ambiguous identity (2 candidate matches)`.
- Action button: **Resolve identity** (dark filled pill).
- On tap, confirmation toast: `Routed to identity resolution — never auto-attached`.

Resolution outcomes (Proposed enumeration of the queue's terminal actions): link to an existing candidate (creates the `matched` mapping and re-runs the event), or create a new Atlas patient then link, or reject the event back to the source. Every identity-linking operation is auditable (PRD §22) — link, unlink, and resolution decisions write audit events with the acting admin.

Acceptance criteria:
- An event whose external ID has an existing `matched` mapping resolves without human involvement and without creating a new patient.
- An event with two candidate matches lands in the human resolution queue in `FAILED` state; no clinical record is attached to any patient until an admin resolves it.
- Resolving creates exactly one `IdentityMapping` row and replays the parked event to completion.
- Every mapping creation/change appears in the audit trail with actor, timestamp, and before/after state.

## 5. Clinical data synchronization (PRD §23)

### 5.1 Supported resources

Demographics · encounters · diagnoses · laboratory orders · laboratory results · imaging orders · imaging reports · prescriptions · medications · procedures · hospital admissions · discharges · discharge summaries · clinical documents · referrals · other supported clinical events (PRD §23). Each tenant connector accepts only its configured whitelist (§3).

Every synchronized record preserves: source system, source record identifier, original clinical timestamp, author/provider, organization, record status, provenance, and version where applicable — and Atlas distinguishes Atlas-originated information from synchronized information (PRD §23).

### 5.2 Ingest pipeline

One logical pipeline for all adapters (Canvas lane 5; Architecture §11.2):

```text
authenticate source → validate schema → resolve patient identity
  → map/normalize → dedupe → persist with provenance
  → apply publication policy → emit events → notify
```

The LLD activity flowchart defines the decision branches. Every failure path terminates in a **visible** state; clinical data is never silently dropped.

| Stage | Guard | On pass | On fail — branch (LLD activity diagram) |
|---|---|---|---|
| 1. Authenticate source | mTLS + OAuth2 client-credentials valid for the tenant connector? | Continue | **Reject `401`** — alert tenant admin, write audit event. Not enqueued; state `REJECTED`. |
| 2. Validate schema | FHIR R4 / HL7 v2 envelope well-formed; resource types declared/whitelisted? | Continue | **`DEAD_LETTER`** — admin error queue with a PHI-minimized reason (demo: "Schema validation failed · missing encounter reference"). |
| 3. Resolve patient identity | Crosswalk/probabilistic match confident? | Continue with `atlas_id` | **Human resolution queue** — never auto-attach (§4.3); surfaced as `FAILED` in the error queue. |
| 4. Map / normalize → dedupe | Idempotency key `{tenant, source_system, source_record_id, version}` new? | Continue as new record | **Acknowledge & discard** — source gets `200`; safe replay, no duplicate record. |
| 5. Persist + provenance | — | Record + provenance stored verbatim (source system, record id, timestamps, author, version) | Transient store failure → `RETRYING` with backoff (§8.2). |
| 6. Publication policy | Tenant rule for this report type says publish? | Continue to events/notify | **Hold** — clinician-only visibility (manual release / restricted report types); record exists but is not patient-visible (§6.2). |
| 7. Emit events + notify | — | `report.finalized` → notify · index · audit (tenant-keyed event, PHI-free push) | Event emission uses the transactional outbox (§7.1), so persist-then-emit cannot half-fail silently. |

Normalization maps source payloads into the internal FHIR-aligned model ([09-data-model.md](09-data-model.md)); transformations never overwrite provenance (§9). Document text extraction and AI embedding also happen at ingest for records feeding ask-the-record retrieval (Architecture §10; see [05-ai-features.md](05-ai-features.md)).

Acceptance criteria:
- Each pipeline stage's failure lands in exactly the branch state in the table; no failure path discards data without a queue entry or an audit record.
- A duplicate delivery is acknowledged with success to the source and provably creates no second record or second notification.
- A synchronized record and an Atlas-originated record are distinguishable via provenance in both API responses and UI.
- A record held by publication policy is visible to authorized clinicians but absent from the patient's timeline and notifications until released.

## 6. Report lifecycle & publication (PRD §24)

### 6.1 Lifecycle states

Modeled so incomplete or preliminary information is never unintentionally presented as final. `Report.status` holds the lifecycle enum, alongside `verified_at`, `published_at`, and `amended_of` (LLD class model).

```text
ORDERED → IN_PROGRESS → COMPLETED → VERIFIED → FINAL → PUBLISHED_TO_PATIENT
```

Atlas typically first sees a report at VERIFIED/FINAL (hospitals push finalized results — §1.3); earlier states exist for sources that publish order/progress events.

Patient- and doctor-facing surfaces render three display states, with the prototype's exact pill styling:

| Display state | Meaning | Pill (prototype) | Demo record |
|---|---|---|---|
| Final | Verified and published as final | Green — bg `oklch(0.93 0.05 150)`, text `oklch(0.42 0.12 150)` | Lipid panel, Apollo Diagnostics, 21 Aug 2026 |
| Preliminary | Not yet final; must not be read as a final clinical report | Amber — bg `oklch(0.93 0.06 80)`, text `oklch(0.45 0.12 60)` | Thyroid panel (TSH, T4), Apollo Diagnostics |
| Amended | A corrected version supersedes an earlier published version | Blue — bg `oklch(0.94 0.02 235)`, text `oklch(0.42 0.12 235)` | Chest X-ray, City General, 10 Nov 2025 |

### 6.2 Publication rules

Only reports meeting the tenant's configured publication criteria become patient-visible. Per report type, a hospital configures one of (PRD §24):

1. **Automatically published after finalization**
2. **Published after verification**
3. **Held for manual release**
4. **Restricted to healthcare professionals** (clinician-only)

Rules live on `TenantOrganization.publication_rules` and are edited in the admin console ("Access policies — Publication rules ›" row on the admin dashboard). Publication also composes with the patient's own access/visibility policy (step 8 of §1.3); consent-category trimming is specified in [07-consent-and-access-control.md](07-consent-and-access-control.md).

On publication: patient notification fires (push copy is PHI-free — LLD sequence: `push "New report available"`; the in-app notification list may name the report: "Lipid panel report is ready · Apollo Diagnostics · 9:12 today"), the search index updates, and the timeline event appears. The sending integration also appears in the patient's access list ("Apollo Diagnostics · Integration · sends reports only · Ongoing").

### 6.3 Amendments

If a report is corrected after publication (PRD §24; Architecture §11.2):

- The amendment creates a **new version**; the prior published version is retained with its provenance.
- The new record links to its predecessor via `amended_of`.
- Published records **visibly indicate that an updated version exists**; lists show the `Amended` pill.
- Proposed: the patient is notified of an amendment through the same PHI-free notification path as initial publication.
- AI artifacts derived from the record are re-embedded on amendment (Architecture §10; see [05-ai-features.md](05-ai-features.md)); side-by-side compare of two versions/results is a doctor-app feature ("Both reports FINAL · Apollo Diagnostics · same fasting protocol" — [03-doctor-app.md](03-doctor-app.md)).

Acceptance criteria:
- A Preliminary report is never rendered with Final styling or described as final in notifications or AI summaries.
- Changing a report type's publication rule affects only subsequent publications; already-published reports remain visible.
- A clinician-only report never appears in the patient timeline, patient notifications, or the patient-scoped AI summary input.
- An amended report retains both versions, links them via `amended_of`, and every surface listing the old version indicates the newer one.

## 7. Event-driven integration (PRD §25)

### 7.1 Event backbone

- Event bus (Kafka-class) with topics including `report.finalized`, `record.created`, `consent.granted`, `consent.revoked`, `access.viewed`, `integration.failed`, `notification.requested` (Architecture §12).
- Producers use the **transactional outbox** pattern — a record persist and its event emission commit atomically.
- Partitions are keyed by `tenant_id`, giving **per-tenant ordering where required** (PRD §25 "event ordering where required"); consumers are idempotent.
- Async consumers: notification dispatch, search indexing, audit projection, AI pre-computation (where enabled), webhook fan-out to tenant systems.
- DLQs are per tenant; events, indexes, caches, and DLQs are all namespaced by tenant (Canvas invariant 5), as are rate limits, quotas, and queues (Canvas invariant 4; Architecture §3.3).

### 7.2 Idempotency, duplicates, replay, acknowledgement

- **Idempotency key**: `{tenant, source_system, source_record_id, version}` computed at ingest. Duplicate events are detected and acknowledged without reprocessing (Architecture §11.2).
- **Safe replay**: because duplicates ack-and-discard with success (`200` to the source — LLD activity diagram), a source can replay any window of events after an outage with no side effects on already-processed records.
- **Source acknowledgement**: synchronous `202 Accepted {event_id}` at ingest (§2.1); the `event_id` correlates later processing status (PRD §25 "source-system acknowledgement", "processing status").
- **Monitoring**: integration lag alerting and DLQ depth alarms are surfaced to tenant admins (Architecture §17 observability; Canvas: "integration lag & DLQ depth alerts to tenant admins"); per-tenant dashboards tag all telemetry with `tenant_id`.

### 7.3 Report-sync sequence (LLD "Sequence · Report sync")

```text
Hospital LIS → Integration GW : 1: POST /integration/fhir/Bundle (mTLS)
Integration GW → Hospital LIS : 2: 202 Accepted {event_id}
Integration GW → Pipeline     : 3: «async» enqueue {idempotency key}
   [pipeline: validate schema · map/normalize to internal model ·
    dedupe on {tenant, source, record_id, version}]
Pipeline → Identity Registry  : 4: resolve(HOSP-928372)
Identity Registry → Pipeline  : 5: ATL-82X92K (confidence 0.98)
Pipeline → Records Svc        : 6: persist(record + provenance)
Records Svc → Pipeline        : 7: stored v1 · status FINAL
Records Svc → Event Bus       : 8: «async» publish report.finalized
Event Bus → Notification      : 9: «async» notification.requested
Notification → Patient App    : 10: push "New report available" (PHI-free)
```

Failures at any async step divert to `RETRYING` or the per-tenant DLQ (§8), visible in the admin error queue.

Acceptance criteria:
- Two events for the same patient from the same source arrive in order at consumers that require ordering (same partition key).
- Replaying a full day of source events after an outage produces zero duplicate records and zero duplicate patient notifications.
- A `report.finalized` event is emitted if and only if the record persist committed (outbox property).
- DLQ depth above threshold and ingest lag above threshold each raise an alert visible to the affected tenant's admins only.

## 8. Integration error handling (PRD §26)

### 8.1 Guarantee

**Failed integrations must not silently lose clinical information.** Every inbound event ends in a terminal state that is either success or *visible* failure ("Fail loudly, never silently" — Architecture principle; error queue subtitle copy: "Failed events never silently drop clinical data.").

### 8.2 Processing state machine

```text
RECEIVED → VALIDATING → PROCESSED
                      ↘ REJECTED
                      ↘ RETRYING → FAILED → DEAD_LETTER
                        (FAILED → DEAD_LETTER by admin disposition only, §8.5)
```

| State | Semantics | Terminal? | Admin-visible? |
|---|---|---|---|
| RECEIVED | Acknowledged (`202 {event_id}`), enqueued with idempotency key | No | Yes (status lookup) |
| VALIDATING | In pipeline stages 2–4 | No | Yes |
| PROCESSED | Persisted (or ack-discarded duplicate); events emitted | Yes | Yes |
| REJECTED | Refused at authentication/authorization (invalid mTLS certificate or OAuth token, or a disabled connector); tenant admin alerted, audited. A non-whitelisted resource type fails schema validation instead and lands in DEAD_LETTER (§5.2 stage 2) | Yes | Yes, with reason |
| RETRYING | Transient failure; retry with exponential backoff. Demo copy: "Endpoint timeout · attempt 3 of 5 · next retry 11:20" (amber pill) | No | Yes, with attempt counter and next-retry time |
| FAILED | Retries exhausted, or a failure needing human action (e.g. ambiguous identity — red pill) | No (admin action can revive) | Yes, with reason and action button |
| DEAD_LETTER | Parked in the per-tenant DLQ; nothing further happens without admin action (gray pill) | Yes (until admin acts) | Yes, with PHI-minimized reason |

Retry policy: exponential backoff; the prototype fixes **5 attempts** (demo shows "attempt 3 of 5"). Proposed schedule: 1 min, 5 min, 15 min, 1 h, 4 h; exhaustion transitions RETRYING → FAILED (admin action can revive — retry, or disposition per §8.5). DEAD_LETTER is never reached automatically on exhaustion: an event parks there only via schema validation failure (§5.2 stage 2) or an explicit admin disposition. Failure reasons shown to admins **avoid unnecessary PHI** (PRD §26; Architecture §11.2: "reasons that avoid unnecessary PHI"; DLQ card: "reason (PHI-minimized)").

### 8.3 Admin dashboard surfacing

On the admin dashboard (City General Hospital), the Integrations section lists each connector with a health dot; a degraded connector renders with a red border (`1.5px solid oklch(0.7 0.14 25)`), red dot (`oklch(0.55 0.18 25)`), and a red count pill (`3 failed ›`, bg `oklch(0.94 0.04 25)`, text `oklch(0.45 0.15 25)`). Tapping it opens the integration queue. Full screen spec: [04-admin-app.md](04-admin-app.md).

### 8.4 Admin integration queue screen

Layout and exact copy (prototype "Admin integration queue"):

- Back link: `‹ Organization`. Title (Instrument Serif 26px): `Legacy HIS — events`. Subtitle: `Failed events never silently drop clinical data.`
- Event cards (white, 14px radius), each with title + status pill + reason line + optional action:

| Card title | Pill | Reason line | Action |
|---|---|---|---|
| `Lab result LAB-102993` | `FAILED` (red) | `HOSP-928372 → ambiguous identity (2 candidate matches)` | **Resolve identity** (dark filled pill) → toast `Routed to identity resolution — never auto-attached` |
| `Discharge summary DS-5521` | `DEAD_LETTER` (gray) | `Schema validation failed · missing encounter reference` | **View payload** (outlined pill) → toast `Payload opened in secure viewer (metadata only)` |
| `Imaging report IMG-8817` | `RETRYING` (amber) | `Endpoint timeout · attempt 3 of 5 · next retry 11:20` | — (automatic) |

- Footer button (full-width, ink `#22303c`): **Retry all recoverable** → toast `2 recoverable events queued for retry` (the RETRYING and FAILED items are recoverable; the schema-invalid DEAD_LETTER item is excluded — it re-fails identically until the payload or mapping changes — hence 2 of 3; see [04-admin-app.md](04-admin-app.md) §4.5).

### 8.5 Admin resolution actions

| Action | Applies to | Effect |
|---|---|---|
| Resolve identity | FAILED (ambiguous identity) | Opens the human resolution queue (§4.3); on resolution the event replays through the pipeline |
| View payload | DEAD_LETTER, FAILED | Secure viewer, metadata-first to minimize PHI exposure; supports diagnosing schema failures |
| Retry / Retry all recoverable | RETRYING, FAILED | Re-enqueues with the original idempotency key (replay-safe); DEAD_LETTER is excluded (schema-invalid payloads re-fail identically until corrected — §8.4) |
| Proposed: Discard with reason | DEAD_LETTER | Terminally closes an event that will never be processed (e.g. source confirms it was sent in error); requires a typed reason; audited. Never a default or bulk action |

All admin queue actions are audited with actor and reason.

Acceptance criteria:
- Every inbound event can be found in exactly one state of §8.2 at any time via its `event_id`; no event is ever unlocatable.
- A transient endpoint timeout retries on the backoff schedule, shows attempt count and next-retry time in the queue, and transitions to FAILED after attempt 5.
- The queue shows failure reasons without exposing report content; payload inspection requires the explicit View payload action and is audited.
- "Retry all recoverable" excludes events requiring identity resolution and never creates duplicates for events that meanwhile succeeded.
- The failed-event count on the dashboard card equals the number of events currently in that connector's error queue (FAILED + DEAD_LETTER + RETRYING — the prototype's `3 failed ›` counts all three queue cards in §8.4).

## 9. Provenance (PRD §27)

Every synchronized clinical record retains its source provenance, stored on the record (`TimelineEvent.provenance`, `version` — LLD class model) and kept **verbatim**: transformation and normalization never overwrite it (PRD §27; Architecture §11.2 "kept verbatim").

```text
Source System:        Hospital EHR            (demo: Apollo LIS)
Source Organization:  Example Hospital        (demo: Apollo Diagnostics)
Source Record ID:     LAB-928372
Source Timestamp:     2026-08-21T10:30:00Z    (original clinical timestamp)
Imported At:          2026-08-21T10:31:02Z
Record Status:        FINAL
```

Plus: author/provider, and version where applicable (PRD §23). Provenance serves auditability, clinical trust, reconciliation, and future interoperability, and is what lets Atlas distinguish synchronized from Atlas-originated records.

Patient-facing rendering — the Report detail screen shows a provenance card (light blue `oklch(0.94 0.02 235)`, 14px radius) with exactly this copy for the demo lipid panel:

```text
Provenance
Source: Apollo LIS · LAB-928372
Verified 21 Aug 2026, 9:02 · Imported 9:03
Status: FINAL · published to patient
```

Amended records keep the provenance of every version; `amended_of` chains versions (§6.3).

Acceptance criteria:
- Every record ingested through any method in §2 has non-null provenance with source system, source organization, source record ID, source timestamp, import timestamp, and record status.
- Re-normalizing or migrating a record leaves its provenance byte-identical.
- The patient report detail renders the provenance card for synchronized reports; Atlas-originated records render their in-app authorship instead.
- Given a record in Atlas, an auditor can name the source system, source record, and both timestamps without access to the source system.

## 10. Out of scope for MVP

Deferred (Architecture §20): full FHIR interoperability surface, HL7 v2 breadth, insurance/pharmacy integrations, dedicated cells and BYOK. The FHIR-aligned model, provenance capture, and the integration boundary are designed so these require no re-architecture. MVP ships one integration path (REST/FHIR ingest of finalized reports), identity mapping, and patient notifications.
