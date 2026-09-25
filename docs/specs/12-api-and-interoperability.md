# 12 — API surface & interoperability

Defines Atlas's versioned REST API surface (conventions, consolidated route catalog, error model, pagination, idempotency), its FHIR R4 alignment, ABDM/ABHA integration points, export formats, and the outbound webhook contract for organization integrations.

Sources: `Atlas_Project_Requirements.md` §10–§16, §19–§29; `Atlas Architecture.dc.html` §2, §5, §6, §11, §12, §15 (API surface); `Atlas LLD.dc.html` (component diagram provided interfaces, sequence diagrams A "QR consult" and B "Report sync", ingest activity flow, domain class model); `Atlas App.dc.html` (prototype interactions the surface must cover); `Canvas.dc.html` (control-plane export, audit export).

Related specs: [06-identity-auth.md](06-identity-auth.md) (token issuance and session model), [07-consent-and-access-control.md](07-consent-and-access-control.md) (policy evaluation semantics), [08-qr-subsystem.md](08-qr-subsystem.md) (QR token lifecycle), [09-data-model.md](09-data-model.md) (entities behind these routes), [11-integrations.md](11-integrations.md) (ingest pipeline behavior), [13-non-functional.md](13-non-functional.md) (SLOs, rate limits).

## 1. API-first principles

PRD §28 and Architecture §2 establish the contract for the whole surface:

1. **Every capability is an API.** The patient app, doctor app, admin console, and hospital integrations are peers of the same versioned surface — no client-private backdoors (Architecture §2.6).
2. **Versioned.** All routes are prefixed `/v1` (Architecture §5). Breaking changes require a new major version; additive changes (new optional fields, new routes) do not.
3. **Authenticated and authorized on every call.** No anonymous access to any resource route. The policy engine (RBAC × ABAC × consent) evaluates every read of clinical data; authentication and authorization are separate concerns (PRD §11, §12; Architecture §7, §14).
4. **Tenant-scoped by construction.** The gateway resolves the tenant from subdomain (`citygeneral.atlas.health`), mTLS client certificate, or token claim and stamps a signed, immutable tenant context `{tenant_id, tier, region, plan}` on the request. `tenant_id` is never accepted from client input; downstream services accept only the signed header (Architecture §3.3, §5). Patient-app calls are platform-scoped (patients are global identities, not tenant members — Architecture §6).
5. **Request schema validation at the gateway**, per-tenant rate limits and quotas, anti-replay nonce checks on QR endpoints, and idempotency keys on writes (Architecture §5).
6. **The surface is built for external integration**: hospitals, diagnostic laboratories, pharmacies, insurance providers, healthcare applications, national/regional health platforms, and EHR systems are all future consumers (PRD §28).

### Acceptance criteria
- Every route in this spec is reachable only under `/v1` and returns `401` without a valid credential.
- A request carrying a client-supplied `tenant_id` (header, query, or body) that conflicts with the derived tenant context is rejected; the derived context always wins.
- The same records API that serves the doctor app can serve a future external EHR consumer without a separate code path.

## 2. Conventions

| Concern | Convention | Source |
|---|---|---|
| Base URL | `https://{tenant-subdomain}.atlas.health/v1` for tenant-scoped clients; `https://app.atlas.health/v1` for the patient app (Proposed hostname split — the subdomain scheme is from Architecture §5) | Architecture §5 |
| Transport | TLS 1.3 only; certificate pinning in mobile apps | Architecture §14 |
| Style | REST, JSON request/response; FHIR-compatible read endpoints planned | Architecture §15 |
| Internal | gRPC service-to-service; REST is the external contract only | Architecture §19 |
| Auth header | `Authorization: Bearer <token>` for users; mTLS + OAuth2 client-credentials token for system clients | Architecture §6 |
| Idempotency | `Idempotency-Key` header on POST/PUT writes (Proposed header name; requirement from Architecture §5) | Architecture §5 |
| Purpose of use | Proposed: `X-Purpose-Of-Use` header on clinical reads (values `TREATMENT`, `EMERGENCY`); recorded in audit and fed to ABAC | Architecture §7, §14 |
| Timestamps | ISO 8601 UTC (`2026-08-21T10:30:00Z`), as shown in PRD §27 provenance example | PRD §27 |
| IDs | Opaque UUIDs in APIs; the human-facing Atlas ID (`ATL-82X92K`) is a display/lookup alias, never an authorization credential | LLD class model; PRD §22 |

### Auth scopes used in the route catalog

| Scope | Principal | Notes |
|---|---|---|
| `patient` | Authenticated patient (or guardian acting for a linked family member) | Platform-level identity; OTP/passkey/ABHA sign-in |
| `doctor` | Practitioner authenticated via org IdP + MFA | Clinical reads additionally require a policy-engine PERMIT (consent) |
| `org-admin` | Organization admin via org IdP + MFA | Tenant-scoped administration |
| `system` | Hospital integration client: mTLS + OAuth2 client-credentials, scoped per tenant | Ingest only; never reads patient-wide data |
| `platform` | Atlas control-plane operators | Tenant lifecycle; no PHI (Architecture §3.1) |

## 3. Route catalog

The API domains are fixed by PRD §28 and Architecture §15: `/auth`, `/patients`, `/doctors`, `/organizations`, `/encounters`, `/diagnoses`, `/prescriptions`, `/medications`, `/reports`, `/documents`, `/hospitalizations`, `/discharge-summaries`, `/consents`, `/access`, `/audit`, `/qr`, `/notifications`, plus `/integration` (tenant-scoped ingest, Architecture §15). The `/admin`, `/ai`, and `/consults` prefixes used below are **Proposed** groupings that do not appear in the PRD §28 / Architecture §15 domain list. Routes named verbatim in the LLD sequence diagrams are marked **[LLD]**; the rest are **Proposed** concretizations needed to cover every prototype interaction and PRD capability within those domains.

The LLD component diagram's provided interfaces map onto this catalog as follows: the API Gateway provides `IPatientAPI`, `IClinicalAPI`, and `IAdminAPI` to the three clients; behind it, `IQRToken` (issue · resolve · revoke · rotate, TTL 5 min) backs §3.6, `IRecords` (timeline · lifecycle · provenance · 360° composition) backs §3.2–3.4, `IConsent` (grants `{categories, duration, purpose}` · revoke · expiry) backs §3.5, and `ISummary` backs §3.7. `IAuthz` (Policy Engine — RBAC × ABAC × consent, break-glass, decision log) and `IAudit` (append-only, hash-chained) are internal service interfaces invoked on every call, not public routes.

### 3.1 Auth (`/auth`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| POST | `/v1/auth/otp` | Request OTP for a mobile number (patient sign-in / registration start). Proposed | public |
| POST | `/v1/auth/otp/verify` | Verify OTP, establish patient session. Proposed | public |
| POST | `/v1/auth/passkey/register` · `/v1/auth/passkey/assert` | Passkey enrolment and sign-in (prototype: "biometric login + passkey"). Proposed | patient |
| GET | `/v1/auth/oidc/{tenant}/authorize` · POST `/v1/auth/oidc/callback` | Doctor/admin federation to the tenant IdP (OIDC/SAML), MFA enforced by IdP policy. Proposed | public → doctor/org-admin |
| POST | `/v1/auth/token` | OAuth2 client-credentials grant for system clients (with mTLS). Proposed | system |
| POST | `/v1/auth/refresh` · POST `/v1/auth/logout` | Short-lived session refresh; sign-out. Proposed | any authenticated |
| POST | `/v1/auth/abha/link` | Link an ABHA address to the patient identity (see §8). Proposed | patient |

### 3.2 Patients & identity (`/patients`, `/doctors`, `/organizations`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| POST | `/v1/patients` | Patient registration (MVP capability, PRD §30). Proposed | public (OTP-verified) |
| GET | `/v1/patients/{id}` | Demographic profile: name, DOB, sex, blood group, allergies, conditions, emergency contact, ABHA address (LLD Patient entity) | patient (self/guardian), doctor (consent-gated) |
| PATCH | `/v1/patients/{id}` | Update patient-editable profile fields (emergency contact, language preference). Proposed | patient |
| GET | `/v1/patients/{id}/summary` | **[LLD seq A, msg 9]** Composed 360° view: overview + AI clinical brief for the authorized slice | doctor (PERMIT required), patient (self) |
| GET | `/v1/patients?query=` | Patient search by name, ID, phone, DOB; results expose **identity only** until an authorization check passes (PRD §15; Architecture §9 Search). Covers the doctor-app search screen. PRD §15's org-specific identifier (hospital MRN via the tenant's IdentityMapping crosswalk) is deferred post-MVP per [03-doctor-app.md](03-doctor-app.md) §5. Proposed route shape | doctor |
| GET | `/v1/patients/{id}/family` | Guardian-linked family members (prototype family switcher: Priya / Aarav / Kamala under one login). Proposed | patient |
| POST | `/v1/patients/{id}/family` | Create a guardian link (family profile under the same login). Proposed | patient |
| GET | `/v1/patients/{id}/emergency-card` | Emergency-card payload (blood group, allergies, conditions, meds, emergency contact) for offline caching on the lock screen; every open is logged (prototype: "Anyone who opens this card is logged"). Proposed | patient |
| GET | `/v1/doctors/{id}` | Practitioner record: name, specialty, tenant, HPR id (LLD Practitioner entity). Proposed | doctor, org-admin |
| GET | `/v1/organizations/{id}` | Tenant organization public profile. Proposed | any authenticated |

Family-member data access: a guardian's calls for a linked member pass the member's patient id; the policy engine authorizes via the guardian link (LLD `Patient.guardian_links`). No separate credential is minted. (Proposed mechanics; the link itself is in the class model.)

### 3.3 Records & timeline (`/encounters`, `/diagnoses`, `/prescriptions`, `/medications`, `/hospitalizations`, `/discharge-summaries`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| GET | `/v1/patients/{id}/timeline` | Unified timeline events (type, occurred_at, provider, organization, provenance, version — LLD TimelineEvent), filterable by `type` (consultations, reports, hospitalizations — prototype filter pills) and date. Proposed route shape | patient, doctor (consent-gated) |
| POST | `/v1/patients/{id}/diagnoses` | Doctor adds a diagnosis (ICD-10 code, status, notes — LLD Diagnosis; prototype "Add to record → Diagnosis"). Proposed | doctor |
| POST | `/v1/patients/{id}/prescriptions` | Doctor adds a prescription (drug, dose, route, frequency, duration — LLD Prescription). Response carries any allergy-conflict warning; saving over a warning requires an explicit `override_reason` and is flagged to compliance (prototype: penicillin-class warning, "Saved with allergy override — flagged to compliance"). Proposed | doctor |
| GET | `/v1/patients/{id}/prescriptions` | Medication list with status Active/Done (prototype Records → Prescriptions tab). Proposed | patient, doctor (consent-gated) |
| GET | `/v1/patients/{id}/hospitalizations` · GET `/v1/hospitalizations/{id}` | Hospitalization episodes incl. procedures and discharge summary (prototype discharge detail screen). Proposed | patient, doctor (consent-gated) |
| GET | `/v1/patients/{id}/observations?code=&from=` | Time-series observations for trend charts (prototype HbA1c/vitals trend: 7.4% → 7.0% → 6.8%). Proposed | patient, doctor (consent-gated) |
| POST | `/v1/patients/{id}/encounters` | Persist the Consultation record for a consult session (reason, notes, related diagnosis/prescription links — [09-data-model.md](09-data-model.md) §6.2); invoked at End consult ([03-doctor-app.md](03-doctor-app.md) §10.1), realizing PRD §3.2 "Record consultations and diagnoses". Proposed | doctor |
| POST | `/v1/encounters/{id}/follow-up/confirm` · `/reschedule` | Follow-up confirm / reschedule-request from the patient notification (prototype: "Follow-up confirmed · added to your calendar", "Reschedule request sent to City General"). Proposed | patient |
| POST | `/v1/consults/{id}/end` | End a QR consult session; closes the session binding and writes the session-end audit event (prototype "End consult" action, toast "Consult ended · 1 session recorded"). Proposed | doctor |

### 3.4 Reports & documents (`/reports`, `/documents`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| GET | `/v1/patients/{id}/reports` | Report list with lifecycle status (Final / Preliminary / Amended visible in prototype). Proposed | patient, doctor (consent-gated) |
| GET | `/v1/reports/{id}` | Single report incl. structured values, reference ranges, provenance, `amended_of` link when a newer version exists (PRD §24). Report compare in the doctor app is a client-side view over two `GET /reports/{id}` calls — no dedicated endpoint. Proposed | patient, doctor (consent-gated) |
| POST | `/v1/patients/{id}/documents` | Doctor/patient document upload: returns a pre-signed upload URL; the document is not visible until malware scanning passes (`scan_status: clean` — Architecture §9, LLD Document). Proposed | doctor, patient |
| GET | `/v1/documents/{id}/download` | Pre-signed, time-limited download URL; download is audited (PRD §13). Proposed | patient, doctor (consent-gated) |

### 3.5 Consent & access (`/consents`, `/access`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| POST | `/v1/consents/requests` | Doctor/org requests access to a patient; triggers a PHI-free notification (Architecture flow C: "Doctor (or org) requests access"; prototype: Dr. Meera Kulkarni's request). Proposed | doctor, org-admin |
| GET | `/v1/patients/{id}/consents` | Patient's grants and requests — the "Who can see my records" access list with Active/Revoked status and expiry (prototype Access screen). Proposed | patient |
| POST | `/v1/consents/requests/{id}/approve` | Approve with `{duration: "24h"\|"30d"\|"90d", categories: {general, reports, medications, mental_health, ...}}` — mental/behavioral health defaults off (Architecture §7; prototype duration picker + category toggles). Proposed | patient |
| POST | `/v1/consents/requests/{id}/deny` | Deny; requester notified (prototype: "Request denied · requester notified"). Proposed | patient |
| POST | `/v1/consents/{grantId}/revoke` | Immediate revocation; policy cache invalidated (Architecture flow C; prototype Revoke action). Proposed | patient |
| POST | `/v1/access/emergency` | Break-glass: `{patient_identifier (Atlas ID, ABHA or phone), reason}` (reason mandatory). Issues a constrained emergency grant limited to the emergency profile + critical history; compliance and the patient alerted immediately; session flagged for review (Architecture flow D; prototype break-glass screen). Proposed | doctor |
| GET | `/v1/patients/{id}/access-log` | Patient-visible access log ("Dr. R. Menon viewed your records · via QR scan") (Architecture §9 Consent & access). Proposed | patient |

### 3.6 QR (`/qr`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| POST | `/v1/qr/tokens` | **[LLD seq A, msg 1]** Issue a fresh rotating QR token for the signed-in patient — opaque, no PHI, TTL 300 s | patient |
| POST | `/v1/qr/resolve` | **[LLD seq A, msg 4]** Body `{token, nonce}`. Resolves token → patient, binds to the scanning doctor's session, consumes the nonce (anti-replay), triggers policy evaluation and consult-grant creation (default duration 30 d — LLD seq A msg 8; prototype access log "access granted (30 days)"), returns the authorized 360° payload reference | doctor |
| POST | `/v1/qr/tokens/{id}/revoke` | Revoke an outstanding token (Architecture §8: tokens revocable). Proposed | patient |
| POST | `/v1/qr/static/resolve` | Printed/static fallback QR — higher-friction flow requiring an additional identity check (Architecture §8). Proposed | doctor |

Full token lifecycle, rotation, and rate limits: [08-qr-subsystem.md](08-qr-subsystem.md).

### 3.7 AI (`ISummary` — LLD AI RAG Service)

All AI routes execute the `authorize → retrieve → assemble → generate → verify → respond` pipeline over the requester's authorized slice only, return citations to source records, and append fixed disclaimers (Architecture §10). Route paths Proposed; capabilities grounded in Architecture §10.1.

| Method | Route | Purpose | Scope |
|---|---|---|---|
| GET | `/v1/patients/{id}/ai/summary` | Plain-language patient home summary; regenerated when new records publish; disclaimer "Generated from N records · not medical advice — ask your doctor" (prototype copy) | patient |
| GET | `/v1/patients/{id}/ai/brief` | Clinical brief at scan time: conditions, latest flagged results, allergies, suggested focus, citations; also invoked internally by `GET /patients/{id}/summary` (LLD seq A msg 10 `generateBrief(authorizedSlice)`) | doctor (PERMIT) |
| POST | `/v1/patients/{id}/ai/ask` | Ask-the-record Q&A (`{question}` → grounded answer + citations; e.g. "HbA1c trend?", "Last eye exam?"). Doctor-only in MVP; a patient-phrased variant is Proposed post-MVP per [05-ai-features.md](05-ai-features.md) §4 | doctor (PERMIT) |
| GET | `/v1/reports/{id}/ai/explain` | Per-report plain-language explanation of values and reference ranges for patients (Architecture §10.1) | patient |
| POST | `/v1/ai/transcribe` | Voice dictation for clinical notes: audio in, transcript out; the transcript is inserted into the note field client-side and saved through the normal record write (prototype dictate action). Proposed — the prototype fixes the feature, not the transport | doctor |

AI output is never written to the record and triggers no clinical action (Architecture §10.5). See [05-ai-features.md](05-ai-features.md).

### 3.8 Notifications (`/notifications`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| GET | `/v1/notifications` | Notification inbox: consent requests, report-ready, access alerts, follow-up reminders (PRD §16; prototype Notifications screen). Proposed | patient, doctor |
| PUT | `/v1/notifications/preferences` | Per-user channel/type preferences (Architecture §9). Proposed | patient, doctor |
| POST | `/v1/notifications/devices` | Register push token; push payloads are PHI-free (Architecture §4). Proposed | patient, doctor |

### 3.9 Integration ingest (`/integration`)

The only door for external clinical data (Architecture §11). Authentication: mTLS + OAuth2 client-credentials, per-tenant (Architecture §6).

| Method | Route | Purpose | Scope |
|---|---|---|---|
| POST | `/v1/integration/fhir/Bundle` | **[LLD seq B, msg 1]** FHIR R4 bundle ingest (finalized reports, diagnoses, prescriptions, discharge summaries, demographics — PRD §23). Returns `202 Accepted {event_id}` synchronously; processing is async | system |
| POST | `/v1/integration/hl7v2` | HL7 v2 message ingest, normalized by protocol adapter (Architecture §11.1). Proposed | system |
| POST | `/v1/integration/events` | Webhook-style event push from source systems (PRD §20, §25). Proposed | system |
| GET | `/v1/integration/events/{event_id}` | Processing status: RECEIVED → VALIDATING → PROCESSED \| REJECTED \| RETRYING → FAILED → DEAD_LETTER (PRD §26) — source-system acknowledgement and monitoring (PRD §25). Proposed | system, org-admin |

File-based (SFTP) and scheduled-sync paths land in the same pipeline; see [11-integrations.md](11-integrations.md).

### 3.10 Admin (`IAdminAPI`)

| Method | Route | Purpose | Scope |
|---|---|---|---|
| GET/POST/PATCH | `/v1/admin/staff` | Staff and role management (Architecture §4 admin console). Proposed | org-admin |
| GET/PUT | `/v1/admin/integrations` · `/v1/admin/integrations/{id}` | Integration configuration: protocol, endpoints, credentials (write-only — never exposed to ordinary users, PRD §21), identifier mappings, supported resources, publication rules, sync mode, webhook config, retry config. Proposed | org-admin |
| GET | `/v1/admin/error-queue` | Failed/DLQ integration events with PHI-minimized failure reasons (PRD §26; prototype error-queue screen). Proposed | org-admin |
| POST | `/v1/admin/error-queue/{id}/retry` · `/retry-all` | Re-queue recoverable events (prototype "Retry all": "2 recoverable events queued for retry"). Proposed | org-admin |
| GET/POST | `/v1/admin/identity-queue` · `/{id}/resolve` | Human resolution of ambiguous identity matches — never auto-attach (PRD §22; LLD activity flow; prototype "Routed to identity resolution"). Proposed | org-admin |
| PUT | `/v1/admin/publication-rules` | Per report type: auto-publish, publish-after-verification, hold-for-manual-release, clinician-only (PRD §24). Proposed | org-admin |
| POST | `/v1/admin/patients/{id}/static-qr` · `/v1/admin/static-qr/{id}/revoke` | Issue / revoke a printed static QR for a patient without a phone (issuance requires recorded in-person identity verification; every operation audited — [04-admin-app.md](04-admin-app.md) §7.7; resolution counterpart is `POST /v1/qr/static/resolve`, §3.6). Proposed | org-admin |
| GET/POST | `/v1/admin/compliance/reviews` · `/{sessionId}/outcome` | Break-glass compliance review queue: list flagged sessions; record a review outcome (justified / unjustified + note) — [04-admin-app.md](04-admin-app.md) §7.6, [07-consent-and-access-control.md](07-consent-and-access-control.md) §6.4. Proposed | org-admin |
| GET | `/v1/audit?patient=&actor=&from=&to=` | Audit query within tenant permissions (PRD §13 domain `/audit`). Proposed | org-admin |
| POST | `/v1/audit/export` | Per-tenant audit export (Canvas: "tamper-evident, per-tenant export"). Proposed | org-admin |

### Acceptance criteria
- Every screen-level interaction in the prototype (QR present/scan, consent approve/deny with duration + categories, revoke, break-glass, add diagnosis/prescription/document with allergy warning, dictation insert, family switch, follow-up confirm/reschedule, error-queue retry, identity resolution, AI chips, report compare) maps to at least one route above.
- `POST /v1/qr/resolve` replays of a consumed nonce are rejected (see §4 error `QR_TOKEN_INVALID`; the replay reason is recorded in audit only, per [08-qr-subsystem.md](08-qr-subsystem.md) §7).
- Patient search responses contain no clinical fields — identity attributes only — regardless of the caller's consent state.
- Prescription writes matching a recorded allergy class return the conflict warning and refuse to persist without `override_reason`.

## 4. Error model

Proposed (the sources fix behaviors — fail loudly, PHI-minimized reasons — but not an envelope shape):

```json
{
  "error": {
    "code": "CONSENT_REQUIRED",
    "message": "Access to this record requires an active consent grant.",
    "trace_id": "req_01J9...",
    "details": [{ "field": "categories", "issue": "mental_health not granted" }]
  }
}
```

| HTTP | Code | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Gateway schema validation failure (Architecture §5) |
| 401 | `UNAUTHENTICATED` | Missing/expired credential; integration auth failure also alerts the tenant admin and is audited (LLD activity: "Reject · 401 · alert tenant admin · audit") |
| 403 | `CONSENT_REQUIRED` | Policy engine DENY for lack of an active grant — clients offer "Request access" |
| 403 | `CATEGORY_EXCLUDED` | Grant active but the requested category (e.g. mental health) is not granted |
| 404 | `NOT_FOUND` | Resource absent **or outside the caller's tenant/consent scope** — cross-tenant probing is indistinguishable from absence (Proposed, from tenancy invariants) |
| 409 | `DUPLICATE` / `VERSION_CONFLICT` | Idempotency-key replay with a different body; stale record version on write |
| 410 | `QR_TOKEN_INVALID` | Token expired, revoked, unknown, or replayed — indistinguishable by design ([08-qr-subsystem.md](08-qr-subsystem.md) §7: failure responses are uniform and information-free, preventing token-space probing); doctor app asks the patient to refresh their QR. Distinct reason codes (expired / revoked / unknown / replay) exist only in the server-side audit trail per 08 §7 |
| 422 | `ALLERGY_CONFLICT` | Prescription conflicts with a recorded allergy and no `override_reason` supplied |
| 429 | `RATE_LIMITED` | Per-tenant / per-doctor limits exceeded; `Retry-After` header |
| 5xx | `INTERNAL` | Never silently drops clinical data — failed ingest lands in RETRYING/DLQ, not a lost 500 (Architecture §2.8) |

Error messages must not leak PHI or cross-tenant existence information (PRD §26 pattern applied surface-wide).

### Acceptance criteria
- All error responses carry the envelope with a machine code and a `trace_id` correlated to observability traces.
- A doctor without consent probing `GET /v1/patients/{id}` for an unknown vs. an unauthorized id cannot distinguish the two responses.

## 5. Pagination, filtering, idempotency

**Pagination (Proposed).** Cursor-based on all list endpoints: request `?limit=` (default 25, max 100) and `?cursor=`; response envelope `{items: [...], next_cursor: "..." | null}`. Cursor pagination keeps timeline reads stable while ingestion appends events. Timeline and reports also accept `?type=`, `?from=`, `?to=`, and `?status=` filters matching the prototype's filter pills.

**Write idempotency.** The gateway supports idempotency keys on writes (Architecture §5). Proposed mechanics: clients send `Idempotency-Key: <uuid>`; the gateway stores the first response for 24 h and replays it for identical retries; a reused key with a different body returns `409 DUPLICATE`. Mobile clients use this on consent decisions, clinical writes, and QR resolution retries.

**Ingestion idempotency (grounded).** The pipeline dedupes on the composite key `{tenant, source_system, source_record_id, version}` (Architecture §11.2; LLD seq B note "dedupe on {tenant, source, record_id, version}"; LLD activity "idempotency {tenant, source, id, version}"). Duplicate events are acknowledged `200/202` and discarded without reprocessing — safe replay for source systems (LLD activity: "source gets 200 — safe replay"). A new `version` for an existing record is not a duplicate: it creates a new record version (amendment), and published records visibly indicate an updated version exists (PRD §24).

### Acceptance criteria
- Replaying the same FHIR bundle twice produces exactly one stored record and two success acknowledgements.
- Replaying a clinical write with the same `Idempotency-Key` returns the original response without a second timeline event.
- Timeline pagination returns no duplicates or gaps while new events are being ingested concurrently.

## 6. FHIR R4 alignment

PRD §29: full FHIR compatibility is not required for MVP, but the model must not make it hard; the internal model is FHIR-aligned (Architecture §9) and FHIR R4 is the first-class ingest protocol (Architecture §11.1) with FHIR-compatible read endpoints planned (Architecture §15).

| Atlas entity (LLD class model) | FHIR R4 resource | Notes |
|---|---|---|
| Patient (atlas_id, demographics, blood group, ABHA address) | `Patient` | `atlas_id` → `Patient.identifier` (Atlas system); ABHA address → identifier with ABDM system URI |
| Allergy (on Patient) | `AllergyIntolerance` | Severity retained ("severe penicillin allergy"); drives the prescribing conflict check |
| Condition (chronic conditions, Diagnosis) | `Condition` | Diagnosis `code: ICD-10` (LLD) → `Condition.code` (ICD-10 coding system) |
| Practitioner (specialty, hpr_id) | `Practitioner` / `PractitionerRole` | `hpr_id` → identifier under the ABDM Healthcare Professional Registry system (Architecture §6 "HPR-ready") |
| TenantOrganization | `Organization` | Tenant/facility hierarchy → `Organization.partOf` |
| TimelineEvent: consultation | `Encounter` | occurred_at, provider, organization |
| Prescription (drug, dose, route, frequency, duration, status) | `MedicationRequest` | Status Active/Done → `active`/`completed`; medication identifiers standardized per PRD §29 (Proposed: coded medication terminology at ingest, free text preserved) |
| Report (lifecycle, verified_at, published_at, amended_of) | `DiagnosticReport` | Lifecycle mapping below; `amended_of` → `DiagnosticReport` version chain |
| Lab values / vitals (trend data) | `Observation` | Proposed: LOINC for laboratory codes (PRD §29 "standardized laboratory codes"); reference ranges in `Observation.referenceRange` |
| Document (mime, storage_ref, scan_status) | `DocumentReference` | Pre-signed URL is the `content.attachment` access path; only `scan_status: clean` documents are exposed |
| Hospitalization (admitted/discharged, procedures, discharge summary) | `Encounter` (class `IMP`) + `Procedure` + `Composition` (discharge summary) | Proposed composition profile for discharge summaries |
| ConsentGrant (categories, duration, purpose, status) | `Consent` | Categories → `Consent.provision.class/category`; expiry → `provision.period`; break-glass → purpose-of-use `ETREAT` (Proposed mapping) |
| Provenance block (source system, org, record id, timestamps, author, version) | `Provenance` | PRD §27 fields map onto `Provenance.agent`/`entity`/`recorded`; never overwritten by transformation |
| AuditEvent (actor, role, tenant, patient, action, resource, method, result, hash_prev) | `AuditEvent` | Hash chain is an Atlas extension; export shape in §9 |
| Guardian link (family profiles) | `RelatedPerson` + `Consent` (Proposed) | Guardian authority modeling |
| Follow-up | `Appointment` (Proposed) | Confirm/reschedule → `Appointment.status` transitions |

### Report lifecycle ↔ FHIR status mapping

Proposed mapping — the Atlas lifecycle states are from PRD §24 and the badges from the prototype; the assignment to `DiagnosticReport.status` values is this spec's concretization:

```text
Atlas lifecycle (PRD §24)          DiagnosticReport.status
ORDERED                        →   registered
IN_PROGRESS                    →   partial
COMPLETED                      →   preliminary
VERIFIED                       →   preliminary        (prototype badge "Preliminary" —
                                                       display compression per 09 §8.3)
FINAL                          →   final              (prototype badge "Final")
PUBLISHED_TO_PATIENT           →   final + Atlas publication flag (extension)
amended after publication      →   amended            (prototype badge "Amended")
```

`PUBLISHED_TO_PATIENT` is an Atlas visibility state, not a FHIR clinical status — Proposed: carried as an extension so round-tripping does not lose publication policy.

### Coding systems

| Domain | System | Status |
|---|---|---|
| Diagnoses | ICD-10 | Grounded (LLD `Diagnosis.code: ICD-10`; persona "E11.9") |
| Laboratory | LOINC | Proposed (PRD §29 asks for "standardized laboratory codes") |
| Medications | Standardized medication identifiers | PRD §29; concrete terminology (e.g. RxNorm or India-appropriate drug dictionary) is an open decision |

### Acceptance criteria
- Every persisted clinical record can be projected to its FHIR resource without loss of the provenance fields in PRD §27.
- A FHIR R4 `DiagnosticReport` bundle from Apollo LIS ingests without custom per-tenant transformation code (mapping via connector configuration only).
- Round-tripping a report through export/ingest preserves lifecycle state and amendment linkage.

## 7. ABDM / ABHA integration points

India-first alignment (approved gap-analysis item; Architecture §6):

- **ABHA address linking.** A patient links their ABHA address to their Atlas identity; the profile shows it with a "Linked" badge (prototype: `priya@abdm · Linked`; welcome screen copy "Works with ABHA (ABDM) · encrypted · you control access"). Route: `POST /v1/auth/abha/link` (§3.1, Proposed mechanics: ABDM-standard OTP/demographic verification against the ABHA service). The ABHA address becomes an additional `Patient.identifier`.
- **ABHA federation sign-in.** Optional patient authentication via ABHA in India deployments (Architecture §6).
- **ABHA as a lookup identifier.** Break-glass and patient search accept "Atlas ID, ABHA or phone" (prototype break-glass input placeholder).
- **Scan-and-share alignment (Proposed roadmap).** Atlas's QR flow is architecturally compatible with ABDM scan-and-share conventions (patient-presented QR resolved by an authenticated professional); adopting the ABDM token/consent-manager protocol is a future-vision item consistent with Architecture §20 ("national platform federation") and PRD §28's national/regional health platforms as future API consumers, and is not in MVP.
- **HPR.** Practitioner verification hook against the Healthcare Professional Registry (`hpr_id` on Practitioner; Architecture §6 "HPR-ready").
- **Residency.** India-deployed tenants pin to the India regional cell; ABDM traffic never leaves it (Architecture §13, §17).

### Acceptance criteria
- Linking an ABHA address requires patient-side verification and is written to the audit log (PRD §22: every identity-linking operation is auditable).
- A doctor can initiate break-glass with an ABHA address as the patient identifier.

## 8. Webhook contract for org integrations

Architecture §15: "Webhooks deliver tenant-facing events with signed payloads." Webhook configuration is part of tenant integration config (PRD §21). Envelope and delivery mechanics below are Proposed; event names are grounded in Architecture §12.

**Subscribable events:** `report.finalized`, `record.created`, `consent.granted`, `consent.revoked`, `access.viewed`, `integration.failed`. Delivery fan-out runs off the event bus (Architecture §12 "webhook fan-out to tenant systems").

**Envelope (Proposed):**

```json
{
  "event_id": "evt_01J9ZK...",
  "event_type": "report.finalized",
  "occurred_at": "2026-08-21T10:31:02Z",
  "api_version": "v1",
  "data": {
    "report_id": "rep_9f2c...",
    "patient_ref": "ATL-82X92K",
    "status": "FINAL",
    "source_system": "Apollo LIS"
  }
}
```

Payloads are PHI-minimized: identifiers and status, never clinical content — the subscriber fetches detail through the authenticated API (consistent with PHI-free notification templates, Architecture §9).

**Delivery rules (Proposed):**

| Rule | Behavior |
|---|---|
| Signature | `Atlas-Signature: t=<unix_ts>, v1=<HMAC-SHA256(secret, t + "." + body)>`; per-endpoint secret held in tenant integration config; secrets rotatable and never shown after creation (PRD §21) |
| Acknowledgement | Any 2xx within 10 s; anything else is a failed attempt |
| Retry | Exponential backoff (aligned with pipeline retry policy, Architecture §11.2), up to 24 h; then the subscription's DLQ — surfaced in the admin error queue, never silently dropped (Architecture §2.8) |
| Idempotency | Consumers dedupe on `event_id`; redeliveries are possible |
| Ordering | Per-tenant partition ordering where required (Architecture §12); consumers must not assume global ordering |
| TLS | HTTPS endpoints only; optional mTLS for dedicated-tier tenants |

### Acceptance criteria
- A subscriber can verify every delivery offline using its endpoint secret; a tampered body fails verification.
- A webhook endpoint that is down for an hour receives all missed events on recovery, in per-tenant order, with no duplicates after `event_id` dedupe.
- Webhook payloads contain no clinical values, note text, or document content.

## 9. Export formats

| Export | Trigger | Format | Source |
|---|---|---|---|
| Patient data export (portable data out) | Patient request | Proposed: FHIR R4 Bundle (JSON) + human-readable PDF summary; documents as original files | Canvas control-plane lifecycle: "export (patient-portable data out)"; PRD §34 "Health data import/export" |
| Tenant audit export | Org-admin request (`POST /v1/audit/export`) | Proposed: NDJSON of FHIR `AuditEvent`-shaped records + hash-chain manifest for tamper verification | Canvas audit service ("tamper-evident, per-tenant export"); Architecture §9 |
| Tenant offboarding export | Control plane offboarding | Full tenant data export, then provable deletion; audit retained per regulation | Architecture §3.3 |
| Report share/download | Patient/doctor | Original document via pre-signed URL (PDF, images, DICOM — Architecture §9); download audited | Architecture §9; PRD §13 |

Full FHIR interoperability surface (a queryable FHIR facade for third parties) is explicitly deferred beyond MVP (Architecture §20); the exports above are file-shaped, not a live FHIR server.

### Acceptance criteria
- A patient export contains every record visible to the patient in-app, with provenance, and validates against FHIR R4 schemas.
- An audit export's hash chain verifies end-to-end; a single altered record breaks verification.
- Offboarding export completes before deletion begins, and deletion is provable while audit retention obligations are honored.

## 10. Open questions

1. Concrete medication terminology for PRD §29 "standardized medication identifiers" (RxNorm vs. an India-appropriate drug dictionary) is undecided in the sources.
2. ~~Whether ask-the-record Q&A is exposed to patients~~ — settled in [05-ai-features.md](05-ai-features.md) §4: doctor-only in the MVP; a patient-phrased variant is Proposed post-MVP. The §3.7 route scope reflects this.
3. Whether dictation transcription runs on-device or via `POST /v1/ai/transcribe` — the prototype fixes only the resulting behavior (transcript inserted into the note field). The governing PHI requirements (on-device preferred, else a PHI-compliant service under BAA/DPA with zero retention, audio never stored) are set in [13-non-functional.md](13-non-functional.md) §2.5; only the concrete transport remains open.
