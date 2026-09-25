# 13 — Non-Functional Requirements

Cross-cutting requirements for security, privacy and compliance, performance, availability, scalability, reliability, observability, accessibility, localization, and operations that every Atlas component must satisfy.

Sources: `Atlas_Project_Requirements.md` (PRD §17 Security, §18 Privacy & Compliance, §31 Non-Functional Requirements, §35 Success Criteria), `Atlas Architecture.dc.html` (§3 Multi-tenancy, §4 Client tier, §5 Edge, §10 AI architecture, §13 Data architecture, §14 Security architecture, §17 Deployment and operations, §18 Reliability and DR), `Canvas.dc.html` (Security & Compliance, Observability, Reliability & DR, Control Plane, Multi-Tenancy Invariants panels), `Atlas App.dc.html` (language selection, offline emergency card, AI disclaimers, QR token copy, Security & sign-in settings).

---

## 1. Scope and cross-references

This spec consolidates the non-functional requirements that apply platform-wide. Functional detail for the mechanisms named here lives in sibling specs:

- Tenant isolation model, service topology, deployment cells — [10-architecture.md](10-architecture.md)
- Authentication mechanisms (OTP, passkey, biometric, org SSO/MFA) — [06-identity-auth.md](06-identity-auth.md)
- Consent semantics, break-glass, policy engine — [07-consent-and-access-control.md](07-consent-and-access-control.md)
- QR token design, anti-replay, session binding — [08-qr-subsystem.md](08-qr-subsystem.md)
- AI pipeline, provider gateway, verification stage — [05-ai-features.md](05-ai-features.md)
- Integration pipeline states, error queue, provenance — [11-integrations.md](11-integrations.md)
- Audit event schema and stores — [09-data-model.md](09-data-model.md)

Where this spec states a target the sibling spec must meet, this spec is authoritative for the number; the sibling spec is authoritative for the mechanism.

---

## 2. Security requirements

PRD §17 states security is "a core requirement rather than a later enhancement." The architecture implements it as five defense-in-depth layers (edge, gateway, service, data, operations). Every requirement below is mandatory for MVP unless marked Proposed.

### 2.1 Encryption in transit

| Requirement | Detail |
|---|---|
| External traffic | TLS 1.3 terminated at CDN/WAF edge; no plaintext HTTP endpoints. |
| Mobile apps | Certificate pinning in both patient and doctor apps. |
| System clients | mTLS for hospital integration clients (in addition to OAuth2 client-credentials). |
| Internal traffic | Proposed: service-to-service traffic encrypted (mesh mTLS or equivalent); the architecture doc specifies gRPC internal transport without stating encryption explicitly. |

### 2.2 Encryption at rest and key management

- AES-256 at rest for every store: clinical DB, identity registry, object storage, search index, audit store, event bus, and the vector store (embeddings are treated as PHI with the same encryption, residency, and deletion obligations as the records they derive from).
- **Per-tenant envelope encryption**: each tenant has its own data encryption keys, wrapped by KMS master keys. The stated invariant: a single leaked key exposes at most one tenant.
- Key rotation is supported for tenant data keys and KMS master keys.
- **BYOK** (customer-managed master keys) is available on the Dedicated tenancy tier only.
- KMS/HSM is the suggested key-custody technology class.
- Tenant provisioning creates the tenant's KMS key automatically as part of the idempotent onboarding automation.
- Cache/queue tier (Redis-class) holds no durable PHI and is TTL-bound; QR token state, sessions, and rate counters live there under tenant-namespaced keys.

### 2.3 Secrets management

- A platform secrets manager holds all credentials. Integration credentials (per-tenant connector secrets, OAuth2 client secrets, SFTP credentials) are stored there and **never exposed to users** (PRD §21: "never be exposed to ordinary users"; architecture: "never exposed to users"). Proposed detail: credentials are write-only in the admin console — tenant admins enter and rotate them but cannot read them back; integrations are configured by reference, not by displaying secrets.
- Proposed: secrets are rotated on a schedule and on suspicion of compromise; no secrets in source control, container images, or environment files; CI secret-scanning enforces this.

### 2.4 Application and session security

- Strong authentication per role (see [06-identity-auth.md](06-identity-auth.md)): patient OTP baseline with passkeys/Face ID once enrolled; doctor org-IdP federation with MFA enforced; system clients mTLS + OAuth2.
- Secure session management: short-lived sessions with refresh; device binding for clinical (doctor) apps.
- Role-based authorization plus fine-grained (category-level, consent-driven) access control evaluated on every read by the policy engine; purpose-of-use recorded on every call.
- Input validation and request schema validation at the gateway; idempotency keys on writes.
- Rate limiting per tenant and per doctor; stricter anti-replay and nonce checks on QR endpoints (see [08-qr-subsystem.md](08-qr-subsystem.md)).
- Secure QR token design: opaque token, no identifiers or PHI in the QR payload, short-lived (minutes), rotating, revocable, session-bound. The patient-facing copy fixed by the prototype makes this a product promise: "This code identifies you — it contains no medical data."
- WAF with DDoS and OWASP filtering plus bot filtering at the edge.
- `tenant_id` is derived server-side (subdomain, client certificate, or token claim) and stamped as a signed, immutable tenant context; downstream services accept only the signed header and never trust client-sent tenant identifiers. Row-level security in Postgres is the backstop, never the primary control.

### 2.5 Mobile and device security

- Certificate pinning in both mobile apps.
- Jailbreak/root detection for clinical (doctor) apps.
- **No PHI in push payloads.** Notification templates are PHI-free by design (e.g. "a new report is available", "Dr. X requested access" — the record contents are fetched only after authenticated open). Proposed detail: push payloads carry only a notification type and an opaque reference ID; the app resolves content after unlock; notification previews on the lock screen never contain diagnoses, medications, or report values.
- Biometric unlock gates locally cached data. The prototype fixes patient-side sign-in options as Face ID and passkey ("Security & sign-in — Face ID + passkey").
- Proposed detail: cryptographic material for passkeys and the biometric unlock key is held in the platform keystore (Secure Enclave on iOS, StrongBox/TEE-backed Keystore on Android); cached PHI on device is encrypted with a key released only on successful biometric or device-credential presentation.
- Offline surfaces are deliberately limited to two: the QR presentation screen and the lock-screen emergency card, both served from cache (see §5.3 graceful degradation). The emergency card is readable without sign-in by design; the prototype's fixed copy discloses the compensating control: "Anyone who opens this card is logged."
- **Screenshot / app-switcher protection (decides the deferral in [02-patient-app.md](02-patient-app.md) §17.4).** Screens rendering PHI — patient timeline, records, report/discharge detail, doctor 360° and add-to-record — are capture-protected per platform: Android sets `FLAG_SECURE` on record screens (blocks screenshots and hides content in the app switcher/recents); iOS blurs or covers the app-switcher snapshot and, Proposed, surfaces a warning on screen-capture detection. The QR present screen is exempt: tokens expire in ≤ 5 minutes, making captures self-defeating ([08-qr-subsystem.md](08-qr-subsystem.md) §4.3), and capture-blocking there would hinder legitimate presentation. The lock-screen emergency card is likewise exempt — it is deliberately shareable safety information. Proposed severity: protection is mandatory on doctor-app clinical screens, default-on for patient record screens.
- **Voice-dictation audio (decides the deferral in [03-doctor-app.md](03-doctor-app.md) §9.1).** Dictation audio is PHI. Requirements: transcription runs **on-device where the platform supports it (preferred)**; otherwise via a PHI-compliant transcription service under the same contractual terms as AI providers (§2.7) — BAA/DPA, zero retention, no training on tenant data. Audio is never stored — neither on-device beyond the transcription buffer nor server-side; only the resulting transcript enters the record, through the normal audited clinical write. The dictation action itself needs no separate audit event; the saved note carries authorship per PRD §13. The concrete transport (on-device vs `POST /v1/ai/transcribe`) remains an open implementation decision ([12-api-and-interoperability.md](12-api-and-interoperability.md) §10), bounded by these requirements.

### 2.6 Secure file handling and malware scanning

- All uploads and downloads go through pre-signed URLs; clients never receive raw storage credentials.
- Every uploaded file passes a **malware-scanning pipeline before the record becomes visible** to any user. Proposed detail: files are quarantined in a staging prefix until scanning passes; a failed scan rejects the upload, notifies the uploader, and writes an audit event; infected files are never served.
- Supported formats: PDF, images, DICOM. Proposed: uploads are validated for declared content type versus actual magic bytes; oversized or malformed files are rejected at the gateway with schema validation.
- Object storage supports lifecycle rules and legal hold.

### 2.7 AI-specific security

(Mechanism detail in [05-ai-features.md](05-ai-features.md); requirements restated here because they are security obligations.)

- Prompt-injection boundary: documents and notes are injected into prompts as quoted data, never as instructions.
- **Red-teaming for prompt injection via uploaded documents** is part of the evaluation harness and gates model/prompt changes alongside clinician-reviewed golden-set regression runs.
- Output filters catch PHI leakage outside the requester's authorized slice; unsupported factual claims are dropped or the response refused (claim verification against retrieved sources).
- Zero data retention and no training on tenant data are hard contractual requirements on every AI provider; no cross-tenant training ever.
- Every generation is logged to audit: requester, slice hash, model, prompt, output — reviewable per tenant, exportable for compliance.
- AI output is never written to the medical record and triggers no clinical action (human-in-the-loop by design).

### 2.8 Operational security

- SAST/DAST in CI, dependency scanning, regular penetration tests.
- Admin (platform-operator) access to production is via just-in-time elevation with audit; **no standing production access to PHI**.
- Comprehensive audit logging of every sensitive action (see §7.2).
- Backup and disaster recovery (§6), data retention policies and secure deletion where legally required (§3.3) complete the PRD §17 checklist.

### Acceptance criteria

- A TLS scan of every public endpoint shows TLS 1.3 with no downgrade path; app builds fail closed on a certificate that does not match the pin.
- Decrypting one tenant's stored data with another tenant's data key fails; rotating a tenant's key does not interrupt reads.
- A request carrying a client-supplied `tenant_id` that conflicts with the resolved tenant context is rejected and audited.
- An uploaded EICAR test file never becomes visible to any user, and the rejection appears in audit.
- No secret value is retrievable through any admin console or API response; secret scanning in CI blocks a seeded canary credential.
- A push notification captured on a locked device contains no name, diagnosis, medication, or report value.
- A crafted document containing prompt-injection text ("ignore previous instructions…") does not alter AI output behavior in the red-team suite.
- Attempted read of cached PHI on a device without biometric/device-credential unlock yields ciphertext only.

---

## 3. Privacy and compliance

PRD §18: Atlas handles highly sensitive personal and medical information; privacy and regulatory compliance are designed in from the beginning, with applicable regulation determined by deployment geography. Canvas states the postures are achieved "via regional cells."

**PRD §18 carries an explicit caveat that this spec preserves as a process requirement: compliance requirements must be validated separately with appropriate legal/compliance expertise before production deployment. The mapping below is an engineering alignment, not legal sign-off.**

### 3.1 Regulatory mapping

Platform controls (all specified elsewhere in this document set) mapped to the three regimes named in the PRD. Rows are the platform controls; the regime columns state the obligation each control serves (obligation phrasing is Proposed as engineering interpretation, per the caveat above).

| Platform control | DPDP (India) | HIPAA (US deployments) | GDPR (EU/EEA users) |
|---|---|---|---|
| Consent as a first-class object; category-level grants; auto-expiry; revocation | Consent-based processing of personal data; withdrawal of consent | Patient authorization for disclosures | Lawful basis (consent); right to withdraw |
| Patient-visible access log; "Dr. X viewed your records" alerts | Notice/transparency to Data Principals | Accounting of disclosures | Transparency; right of access to processing information |
| Data export (patient-portable data out; per-tenant export at offboarding) | Data Principal access rights | Individual right of access to PHI | Right of access; data portability |
| Secure deletion honoring audit-retention obligations; provable deletion at tenant offboarding | Erasure once purpose is served | Media disposal / retention rules | Right to erasure, balanced against legal retention |
| Regional cells; per-tenant residency pin; backups and replicas stay in-region | Data-localization posture for India deployments | n/a (residency by contract) | Transfer restrictions; EU-resident processing |
| AES-256 at rest, TLS 1.3 in transit, per-tenant envelope keys | Reasonable security safeguards | Security Rule technical safeguards | Security of processing (Art. 32-class) |
| Append-only hash-chained audit, WORM retention | Accountability | Audit controls | Accountability principle |
| Breach alerting paths (compliance alerted on break-glass; DLQ/incident alerting) | Breach notification duty | Breach notification | Breach notification |
| AI provider BAA/DPA, zero retention, no training on tenant data | Processor obligations | Business Associate Agreements | Processor contracts (Art. 28-class) |

- India-first alignment: ABHA (ABDM) federation is supported for patient identity, and the practitioner registry is HPR-ready.
- Per-provider BAA/DPA contracts are a hard requirement for AI providers; requests to AI providers carry no tenant identifiers beyond what the contract covers.

### 3.2 Data residency

- Each tenant is pinned to a regional cell (e.g. India, EU, US) at provisioning; the pin covers compute, primary stores, **backups, and replicas** — residency never crosses cells.
- Regional load balancing at the edge honors the tenant's residency pin.
- In-region failover preserves residency during DR (see §6).
- AI model routing is per-tenant configuration including region; Dedicated/government cells can run self-hosted models with no egress.
- Dedicated tier supports a pinned region as a contractual feature; required for national-platform deployments.

### 3.3 Retention and deletion

- Retention is **policy-driven per record class and jurisdiction** — the retention schedule is configuration, not code.
- Secure deletion is supported where legally required, and always honors audit-retention obligations: deleting clinical content does not delete the audit trail of its lifecycle.
- Object storage supports lifecycle rules and legal hold for documents.
- Tenant offboarding: export the tenant's data, then **provably delete** it; audit records are retained per regulation. Proposed detail: "provably" means a deletion manifest enumerating deleted objects/keys, plus destruction of the tenant's data encryption keys (crypto-shredding), delivered to the departing tenant.
- Search indexes and vector stores are disposable and rebuilt from source of truth; deletion of a source record must propagate to derived stores (index entries, embeddings, cached summaries). Embeddings explicitly carry the same deletion obligations as source records.

### 3.4 Patient rights surfaces

These are product requirements, not just policy statements — each has a UI surface in the patient app:

| Right | Surface |
|---|---|
| See who accessed my data | Patient-visible access log (Access & consent screen); push alert on every doctor view; break-glass access alerts the patient immediately. |
| Control access | Consent grant/deny with duration (24 h / 30 d / 90 d) and per-category toggles; mental/behavioral health defaults off. |
| Revoke access | Revocation available at any time; policy cache invalidation makes revocation take effect immediately, including on AI retrieval (authorization-filtered at query time). |
| Export my data | Patient-portable data out (control-plane lifecycle capability). Proposed: patient-initiated export from the app ships post-MVP; MVP honors export requests via support/tenant process. |

### Acceptance criteria

- Provisioning a tenant pinned to the India cell results in zero PHI bytes stored or backed up outside that cell (verifiable from storage inventory).
- A revoked consent grant blocks the next read and the next AI generation for that grantee with no cache serving stale authorization beyond the seconds-scale policy-cache TTL.
- Offboarding a test tenant produces a complete export, then leaves no readable tenant data in any store while its audit stream remains intact and readable.
- Deleting a record removes its search-index entries, embeddings, and any cached summary derived from it.
- Every break-glass access generates an immediate patient alert and a compliance alert, and is flagged in audit for review.
- A documented legal/compliance review sign-off exists before any production deployment (process gate from PRD §18).

---

## 4. Performance

PRD §31 sets the qualitative bar — QR-to-patient lookup "near-instantaneous," patient overview loads quickly "even for patients with extensive history," document access supports large files "without blocking the main application." The architecture doc quantifies the critical paths.

### 4.1 Latency SLOs

| # | Path | Target | Source |
|---|---|---|---|
| P1 | QR-to-patient lookup (scan → identified patient) | p95 < 500 ms | Architecture §17, Canvas Observability |
| P2 | Patient 360° overview load (authorized slice composed) | < 1 s | Architecture §17, Canvas Observability |
| P3 | AI clinical brief at scan time | p95 < 2.5 s | Architecture §10.4 |
| P4 | Document access (large files, incl. DICOM) | Must not block the main application: pre-signed URL issued fast, transfer is direct-to-storage and async from the app's perspective | PRD §31, Architecture §9 |
| P5 | Standard authenticated API reads | Proposed: p95 < 300 ms in-region | — |
| P6 | Patient search (doctor-side, identity-only results) | Proposed: p95 < 800 ms | — |
| P7 | Ask-the-record Q&A answer | Proposed: p95 < 4 s (interactive chat tolerance; brief target P3 does not apply) | — |

Notes on the QR consult path (flow A): P1 covers gateway tenant resolution + rate/nonce check + token resolution + session binding. P2 covers policy evaluation plus cross-tenant, consent-trimmed composition (read replicas exist specifically for heavy patient-360 queries). P3 runs from the same authorized slice; tenants may enable pre-computation of briefs on `report.finalized` to hit the target, with the cache keyed by `{patient, authorized-slice hash, model version}` and invalidated by new records, consent changes, or model upgrades.

### 4.2 Perceived performance requirements

- The doctor app must render the 360° view progressively: identity and safety-critical facts (allergies, conditions) must not wait for the AI brief; the prototype shows the brief loading asynchronously after the record view ("aiReady" state) with the record usable meanwhile.
- The patient QR screen must display instantly from cache (offline-capable) while a fresh rotating token is fetched; the visible countdown ("Secure token · rotates in 4:52") reflects token freshness.
- Proposed: timeline and report lists paginate/virtualize; the "extensive history" requirement (PRD §31) is met by composing the overview from summaries + latest results, not by loading the full history eagerly.

### Acceptance criteria

- Load tests demonstrate P1–P3 at production-like data volumes (a patient with multi-year, multi-tenant history) and at per-tenant rate-limit saturation.
- SLO dashboards exist per tenant for P1 and P2 (the two per-tenant SLOs named by the architecture and canvas; contractual custom SLOs are a Dedicated-tier feature per §5.1); alerting fires on burn-rate breach.
- A 500 MB DICOM download neither blocks nor degrades concurrent API interactions in the same app session.
- Disabling AI pre-computation degrades P3 gracefully (brief arrives later with a loading state) without affecting P1/P2.

---

## 5. Availability and scalability

### 5.1 Availability

- PRD §31: high availability is required because doctors depend on Atlas **during active patient care**.
- Multi-AZ by default in every regional cell; blue/green or canary deploys mean releases take no downtime; schema migrations are gated and reversible.
- Proposed numeric targets (no figure appears in the sources; these operationalize "high availability" and must be ratified): 99.9 % monthly availability for the clinical read path (QR resolve, 360° view, document read) platform-wide; custom SLOs are a Dedicated-tier contractual feature per Canvas ("custom SLOs"). Availability is measured per cell and reported per tenant.

### 5.2 Degradation order

Reads outlive writes: if the platform is degraded, the read path (doctor viewing records) is preserved in preference to the write path (new clinical data, integrations). Integration ingest can lag and queue safely (§6.2); the consult path cannot.

### 5.3 Graceful degradation (explicit offline/degraded surfaces)

| Surface | Behavior when core services are degraded/unreachable |
|---|---|
| Patient QR screen | Served from device cache; last-issued token displayed. (Token expiry still applies; Proposed: the app shows token staleness rather than silently presenting an expired token.) |
| Lock-screen emergency card | Fully offline by design — name, age/sex, blood group B+, severe allergies, conditions, medications, emergency contact, Atlas ID; readable with no sign-in; every open logged (copy fixed by prototype). |
| Doctor 360° after load | Proposed: already-composed view remains readable; writes queue or fail visibly, never silently. |

### 5.4 Scalability model

- PRD §31 requires independent scaling of: API services, database, document storage, authentication, search, audit logging. The architecture meets this with stateless services and isolated state: horizontal autoscaling per service, with API, records, integration workers, and AI scaling independently.
- **Tenancy tiers** scale isolation with customer size (same code, different isolation):

| Tier | Compute | Database | Storage/search | Customer |
|---|---|---|---|---|
| Pooled | Shared | Shared tables + `tenant_id` + RLS | Per-tenant prefixes/aliases | Clinics, small labs |
| Siloed | Shared, optional dedicated queue consumers | Schema-per-tenant on shared clusters | Dedicated bucket + index partition | Hospitals |
| Dedicated | Dedicated cell | Own cluster, pinned region, optional BYOK | Fully dedicated | Networks, government |

- **Noisy-neighbor control**: per-tenant rate limits and quotas enforced at the gateway; queues, indexes, caches, and DLQs all namespaced by tenant; noisy-neighbor detection in observability is tied to quota enforcement. Per-tenant usage metering (API calls, storage, seats) feeds the control plane.
- Clinical DB is partitioned by tenant with read replicas for patient-360 composition; event bus partitions are keyed by `tenant_id` with per-tenant ordering where required.
- Tenant onboarding is automated and idempotent — "minutes not days" — so scale-out to new tenants is an operational non-event.
- MVP boundary: pooled + siloed tiers in one region; dedicated cells and BYOK are deferred without re-architecture.

### Acceptance criteria

- Killing one AZ in a cell causes no data loss and keeps the clinical read path available.
- A pooled tenant driven to its rate limit degrades only its own traffic; co-tenant P1/P2 SLOs hold (verified by a noisy-neighbor load test).
- Each of the six PRD-listed subsystems can be scaled independently without redeploying the others.
- Airplane-mode test: QR screen and emergency card render fully with no connectivity.

---

## 6. Reliability and disaster recovery

### 6.1 Durability, backup, DR

| Requirement | Target / mechanism |
|---|---|
| RPO | ≤ 5 minutes |
| RTO | ≤ 1 hour for clinical reads |
| Backups | Point-in-time-recovery backups per tenant DB/schema; in-region |
| Restore validation | Regular restore drills (not just backup success) |
| Failover | In-region (multi-AZ) failover preserves residency |
| Integrity | Data integrity checks on clinical records; audit hash chain anchored daily |
| Migrations | Safe migration mechanisms: gated, reversible schema migrations (PRD §31 + Architecture §17) |

Durable medical records are a PRD reliability requirement; derived stores (search, vectors, caches) are explicitly disposable and rebuilt from source of truth, so DR scope concentrates on the clinical DB, identity registry, object storage, and audit store.

### 6.2 No-silent-drop guarantee

PRD success criterion 16: "Integration failures are detectable, retryable, and do not silently lose clinical information." Architecture principle 8: "Fail loudly, never silently."

- Every inbound clinical payload lands in a visible state: `RECEIVED → VALIDATING → PROCESSED | REJECTED | RETRYING → FAILED → DEAD_LETTER` — all states visible to tenant admins with failure reasons that avoid unnecessary PHI.
- Retry with exponential backoff; per-tenant dead-letter queues; DLQ depth alarms surface to tenant admins.
- Idempotency key `{tenant, source_system, source_record_id, version}` — duplicates acknowledged without reprocessing, so retries are safe.
- Ambiguous identity matches route to the admin resolution queue — **never auto-attach** a record to a possibly-wrong patient.
- Producers use the transactional outbox pattern; consumers are idempotent — events are not lost between service write and bus publish.

### Acceptance criteria

- A game-day restore from PITR backup meets RPO ≤ 5 min / RTO ≤ 1 h for clinical reads, per tenant tier, evidenced by drill records.
- Injecting a poison message into an integration connector results in a DEAD_LETTER entry visible in the admin error queue with a reason — zero payloads unaccounted for across a chaos run.
- Replaying the same source event (same idempotency key) creates no duplicate clinical record.
- Daily audit hash-chain anchor verification detects a deliberately tampered test event.

---

## 7. Observability

### 7.1 Telemetry

- Structured logs, metrics, and traces — all tagged with `tenant_id` (OpenTelemetry is the suggested stack).
- Per-tenant dashboards and SLOs: QR-to-patient p95 < 500 ms and overview load < 1 s are the named per-tenant SLO panels; integration lag alerting and DLQ depth alarms are surfaced to tenant admins in the admin console.
- Noisy-neighbor detection tied to quota enforcement.
- AI operations metered per tenant (token metering feeds control-plane billing); model/provider failover and circuit-breaker state are operational signals.
- Proposed: telemetry (logs, metrics, traces) must be PHI-free — identifiers only (tenant, actor, resource IDs), never clinical content; this keeps the observability stack outside the PHI compliance boundary.

### 7.2 Audit versus telemetry separation

Audit and telemetry are different systems with different guarantees and must not be conflated:

| Dimension | Audit | Telemetry |
|---|---|---|
| Purpose | Legal/compliance record of who did what to whose data | Operating the platform |
| Content | Every view, write, grant, QR scan, break-glass, export: `{actor, role, tenant, patient, action, resource, method, result, session}`; AI generations (requester, slice hash, model, prompt, output) | Latency, errors, saturation, request traces |
| Store | Append-only WORM, hash-chained, anchored daily, immutable retention | Standard observability stack, operational retention |
| Access | **Separate credentials from app DBs**; per-tenant export; patient-visible projection (access log) | Engineering/on-call access |
| Loss tolerance | None — audit write is part of the sensitive operation | Sampling acceptable (Proposed) |

### Acceptance criteria

- Any request can be traced end-to-end by trace ID with `tenant_id` on every span.
- Compromise of observability-stack credentials grants no read or write access to the audit store (separate credential domains, verified in access review).
- A scan of exported telemetry for a seeded synthetic patient shows no clinical values or names.
- Tenant admins can see integration lag and DLQ depth for their tenant only.

---

## 8. Accessibility and localization

### 8.1 Localization

- Supported languages at MVP: **English, Hindi (हिंदी), Marathi (मराठी)** — fixed by the prototype's welcome-screen language chips and the profile "Language" setting. India-first deployment (ABDM/ABHA alignment).
- Scope: all patient-app UI copy, notification templates, and AI patient-facing summaries. Proposed: the doctor app and admin console ship English-only at MVP (clinical/professional audience; no non-English doctor-side copy appears in the prototype); clinical record content is displayed in its source language, untranslated.
- Proposed: AI patient summaries and report explanations are generated in the patient's selected app language; disclaimers are translated fixed strings, not model output.
- Devanagari rendering must be first-class in the app fonts; Instrument Sans/Serif lack Devanagari, so a companion Devanagari face is required (Proposed: a matched fallback stack per [01-design-system.md](01-design-system.md)).
- Locale is also a per-tenant branding/config concern in the control plane ("logo/locale per tenant").

### 8.2 Accessibility (Proposed — the sources fix visual design but state no accessibility standard; this section makes the implied bar testable)

- Conformance target: WCAG 2.1 AA for all three clients.
- **Font scaling**: support OS-level Dynamic Type / font scale up to 200 % without loss of content or function; layouts reflow rather than truncate. Safety-critical values (allergy "Penicillin — severe", blood group "B+") must remain visible unclipped at maximum scale.
- Color is never the sole carrier of meaning: report states (Final/Preliminary/Amended), allergy severity, and trend flags (TG 168 ↑) pair color with text/symbols — the prototype already does this; implementations must preserve it.
- Contrast: ink `#22303c` on paper `#faf7f1` and the ocean-blue accent must meet AA ratios at the sizes used; verify the low-emphasis grays (`rgba(34,48,60,0.45–0.6)`) against AA for body-size text and adjust where they fail.
- Full screen-reader support (VoiceOver/TalkBack): QR screen announces token freshness; charts (HbA1c/vitals trends) expose data as text alternatives.
- Touch targets ≥ 44 pt/48 dp; the consult flow (scan → view → prescribe) operable without precise gestures.
- Voice dictation for clinical notes (an approved feature) doubles as an accessibility affordance and must not be the only input path.

### Acceptance criteria

- Switching language re-renders the entire patient app, notifications, and AI summaries in Hindi or Marathi with no untranslated MVP strings.
- The app at 200 % font scale passes a scripted walkthrough of all thirteen patient screens in the prototype with no clipped safety-critical data.
- Automated accessibility audit (axe-class) plus manual VoiceOver/TalkBack pass on the QR, emergency card, consent, and report screens.

---

## 9. Operational requirements

### 9.1 Delivery and change management

- Trunk-based CI/CD; blue/green or canary deploys; per-tenant feature flags for progressive rollout.
- Schema migrations gated and reversible; model/prompt changes for AI gated by evaluation-harness regression runs; model version pinning with staged upgrades.
- SAST/DAST and dependency scanning are CI gates (§2.8).

### 9.2 On-call and incident response (Proposed — sources establish the alert signals, not the human process)

- 24×7 on-call for the clinical read path, because the availability requirement is anchored to active patient care.
- Paging severities: SEV-1 = clinical read path (QR resolve, 360° view) impaired in any cell; SEV-2 = write/integration path impaired or DLQ growth breaching SLA; SEV-3 = single-tenant degradation within quota.
- Break-glass usage review and audit-chain anchor failure are compliance-notified events with a defined review turnaround, not just engineering pages.

### 9.3 Error-queue and integration SLAs (Proposed — quantifies "visible, never dropped")

| Signal | SLA |
|---|---|
| Integration lag (finalized at source → visible per publication policy) | Alert at 15 min; SEV-2 at 60 min |
| DLQ item triage by tenant admin surface | New DEAD_LETTER items visible in the admin error queue within 1 min of final failure |
| Identity-resolution queue (ambiguous matches) | Reviewed within 1 business day; records remain unattached, never auto-attached, until resolved |
| Retry policy | Exponential backoff, bounded attempts; transition to FAILED/DEAD_LETTER is always explicit and audited |

### 9.4 Tenant operations

- Tenant provisioning: automated, idempotent, minutes not days (schema/DB, storage prefix, KMS key, search partition, IdP config, subdomain, quotas).
- Sandbox connector for integration testing before go-live (test events flow end-to-end without touching production data).
- Lifecycle operations: suspend, export, offboard with secure deletion and retention holds — all control-plane capabilities.

### Acceptance criteria

- A canary deploy that fails health checks rolls back automatically with no tenant-visible downtime on the read path.
- A simulated integration outage pages within the lag SLA and the resulting backlog drains without manual replay (idempotent consumers).
- A new tenant reaches a working sandbox integration in under one day using only self-serve provisioning plus admin-console configuration.

---

## 10. Compliance verification gate

Restating the PRD §18 process requirement as a release gate: before any production deployment in a new geography, (1) the applicable regulation set for that geography is identified, (2) the §3.1 mapping is reviewed against it by qualified legal/compliance counsel, and (3) sign-off is recorded. Engineering controls in this spec are designed to make that review pass, but do not substitute for it.
