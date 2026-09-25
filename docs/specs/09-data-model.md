# 09 — Domain data model

Defines every persistent entity in the Atlas platform — attributes, relationships, lifecycles, ownership, and the timeline event taxonomy — as the single reference for schema design and API contracts.

Sources: `Atlas LLD.dc.html` (Domain model class tab, Activity · Ingest tab, both sequence tabs), `Atlas_Project_Requirements.md` §4–9, §22–27, `Atlas Architecture.dc.html` §3, §7, §9, §11–13, `Canvas.dc.html` (multi-tenancy invariants), `Atlas App.dc.html` (sample data, status pills, provenance block, admin error queue).

Related specs: storage/tenancy in [10-architecture.md](10-architecture.md), consent semantics in [07-consent-and-access-control.md](07-consent-and-access-control.md), ingest pipeline in [11-integrations.md](11-integrations.md), API resources in [12-api-and-interoperability.md](12-api-and-interoperability.md), UI presentation of these entities in [02-patient-app.md](02-patient-app.md), [03-doctor-app.md](03-doctor-app.md), [04-admin-app.md](04-admin-app.md).

## 1. Modeling principles

1. **FHIR-aligned, not FHIR-bound.** The internal model uses FHIR-compatible shapes and vocabulary (ICD-10 codes on Diagnosis, FHIR R4 accepted at the integration boundary) without requiring full FHIR conformance in MVP (PRD §29; Architecture §9, §20).
2. **Patient identity is global; clinical records are tenant-owned.** One Atlas Patient ID spans tenants. Every clinical record carries the `tenant_id` of the organization that produced it. The 360° view is a consented, read-time composition across tenants — never a copy (Architecture §1, §3; Canvas invariant 6).
3. **TimelineEvent is the clinical backbone.** All clinical happenings are subtypes of one abstract event, so the timeline, filters, search, and AI retrieval operate over a single polymorphic stream (LLD class model).
4. **Provenance is immutable.** Synchronized records keep their source provenance verbatim; transformations and normalization never overwrite it (PRD §27; Architecture §11.2).
5. **Audit references, never owns.** `AuditEvent` points at patients and resources but holds no clinical payload, and lives in a separate append-only store (LLD class-tab description: "AuditEvent references, never owns, clinical entities").
6. **Nothing clinical is silently dropped.** Failed integrations become visible `IntegrationEvent` records with explicit terminal states (PRD §26; LLD activity diagram).

## 2. Entity inventory and ownership

| # | Entity | Scope | Owning store (Architecture §13) | Source |
|---|--------|-------|--------------------------------|--------|
| 1 | Patient | **Global** (platform) | Identity registry (PostgreSQL, service-only access) | LLD class model («global» on `atlas_id`) |
| 2 | TenantOrganization | Control plane | Control-plane DB (no PHI) | LLD class model; Architecture §3.1 |
| 3 | Practitioner | Tenant-owned | Tenant directory | LLD class model; Architecture §6 |
| 4 | ConsentGrant | Patient-scoped, platform-evaluated | Clinical DB (Consent & access service) | LLD class model; Architecture §9 |
| 5 | IdentityMapping | Per-tenant crosswalk onto global identity | Identity registry | LLD class model; PRD §22 |
| 6 | TimelineEvent (abstract) + subtypes | Tenant-owned | Clinical DB (RLS / schema / cluster by tier) | LLD class model |
| 7 | Report (TimelineEvent subtype) | Tenant-owned | Clinical DB | LLD class model; PRD §24 |
| 8 | Document | Tenant-owned | Object storage (per-tenant bucket/prefix) + metadata in Clinical DB | LLD class model; PRD §6 |
| 9 | Provenance (value object) | Embedded in TimelineEvent | Clinical DB | LLD class model; PRD §27 |
| 10 | AuditEvent | Per-tenant stream, platform-operated | Audit WORM store (hash-chained) | LLD class model; PRD §13 |
| 11 | IntegrationEvent | Tenant-owned | Event bus + per-tenant DLQ; queue state visible to admins | PRD §25–26; Admin UI |
| 12 | FamilyLink | Global (between Patients) | Identity registry | Patient.`guardian_links`; prototype family switcher |

Derived data (search indexes, per-tenant embeddings/vector store, AI summary cache) is not part of the domain model: it is disposable, rebuilt from these entities, and vectors are treated as PHI (Architecture §10.4, §13). See [10-architecture.md](10-architecture.md).

**Acceptance criteria**

- Every clinical table carries a non-null `tenant_id`; the Patient and FamilyLink tables carry none.
- A query for a patient's cross-tenant timeline succeeds only through the consented-composition path and produces an AuditEvent (Canvas invariant 6).
- Deleting a tenant (offboarding) removes all rows scoped by that `tenant_id` except audit records retained per regulation (Canvas invariant 7).

## 3. Patient (global)

The golden demographic record. One per human; `atlas_id` is the identifier encoded in the patient's ID string (display form `ATL-82X92K`) and resolved by QR tokens (see [08-qr-subsystem.md](08-qr-subsystem.md)).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `atlas_id` | UUID | ✔ | «global» — platform-wide, tenant-independent (LLD). Display alias `ATL-XXXXXX`. |
| `name` | string | ✔ | Full name (PRD §4). |
| `dob` | date | ✔ | Drives displayed age ("34 F"). |
| `sex` | enum | ✔ | LLD `sex`; PRD §4 "Gender". |
| `blood_group` | enum | ✔ | e.g. `B+`; shown on profile and emergency card. |
| `allergies` | Allergy[] | – | Structured list (LLD `allergies: Allergy[]`). See §3.1. |
| `conditions` | Condition[] | – | Structured list (LLD `conditions: Condition[]`). See §3.1. |
| `emergency_contact` | object | – | Name + phone ("Aman S. · +91 98111 20844"). |
| `abha_address` | string? | – | Optional ABHA (ABDM) health address, e.g. `priya@abdm`; profile shows a "Linked" badge when present (India deployments; Architecture §6). |
| `guardian_links` | Link[] | – | FamilyLink references (§14). |
| `contact`, `address`, other demographics | object | – | PRD §4 basic information. Proposed: phone (`+91 98220 41175` is the OTP sign-in number), email, address as structured fields. |

Rules:

- Patients are **platform-level identities, not tenant members** (Architecture §6). No `tenant_id`.
- Duplicate prevention: new registrations and integration ingests must attempt a confident match against existing patients before creating a record (PRD §22).
- Sensitive fields are exposed only per the caller's authorization and access policy (PRD §4).

### 3.1 Allergy and Condition (embedded value objects)

The prototype fixes the display shapes: allergy "Penicillin — severe reaction (2019)" and condition "Type 2 diabetes · 2022" / "Type 2 diabetes (E11.9)".

| Allergy field | Type | Notes |
|---------------|------|-------|
| `substance` | string | "Penicillin". Drives the prescribing conflict check (penicillin-class match incl. amoxicillin, augmentin — see [03-doctor-app.md](03-doctor-app.md)). |
| `severity` | enum | "severe" shown; Proposed levels: `mild | moderate | severe`. |
| `recorded_year` | int | "(2019)". |

| Condition field | Type | Notes |
|-----------------|------|-------|
| `name` | string | "Type 2 diabetes", "Allergic rhinitis". |
| `code` | string? | ICD-10 where known ("E11.9"). |
| `since_year` | int? | "· 2022". |

**Acceptance criteria**

- A patient created in one tenant's context is immediately resolvable by any other tenant via QR/consent without duplication.
- Profile, emergency card, and doctor-360 header all render from these fields (name, age/sex, blood group, allergies, conditions, emergency contact, ABHA state) with no per-screen copies.

## 4. TenantOrganization

A healthcare organization (hospital, clinic chain, diagnostics network). Tenant of the SaaS; owns the clinical records it produces (Architecture §1).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `tenant_id` | UUID | ✔ | Derived server-side on every request — never from client input (Canvas invariant 1). |
| `name` | string | ✔ | "City General Hospital", "SkinCare Clinic", "Apollo Diagnostics". |
| `type` | enum | ✔ | LLD `type`; Proposed values: `hospital | clinic | lab | network`. |
| `tier` | enum | ✔ | `pooled | siloed | dedicated` (LLD; Architecture §3.2 — decides RLS vs schema vs cluster isolation). |
| `region` | string | ✔ | Regional cell pin; backups/replicas stay in-region (Architecture §13). |
| `idp_config` | object | – | OIDC/SAML federation for staff sign-in (Architecture §6). |
| `quotas` | object | – | Per-tenant rate limits/quotas (Canvas invariant 4). |
| `publication_rules` | object | – | Per report type: auto-publish after finalization, publish after verification, hold for manual release, or restrict to clinicians (PRD §24; LLD). Surfaced in Admin → "Access policies · Publication rules". |

**Acceptance criteria**

- Changing `tier` or `region` is a control-plane operation and never mutable through data-plane APIs.
- `publication_rules` changes affect only future publication decisions; already-published reports are unaffected (Proposed).

## 5. Practitioner

A doctor or staff member, employed by exactly one tenant (LLD "employs" association).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `practitioner_id` | UUID | ✔ | |
| `tenant_id` | FK → TenantOrganization | ✔ | Doctors belong to tenants and sign in through the tenant's IdP (Architecture §1, §6). |
| `name` | string | ✔ | "Dr. Rohan Menon". |
| `specialty` | string | ✔ | "Endocrinology", "Dermatology". Shown to patients in consent requests. |
| `hpr_id` | string? | – | Professional-registry identifier (HPR-ready hook; Architecture §6). Consent screen shows "verified professional". |
| `role` | enum | ✔ | LLD `role`; RBAC roles: doctor, org-admin (+ platform roles) (Architecture §7). |
| `mfa_enrolled` | boolean | ✔ | MFA is enforced for clinical sign-in (Architecture §6). |

A practitioner who moves organizations is a new Practitioner row under the new tenant; their historical authorship in the old tenant's records is untouched (Proposed, follows tenant-ownership rule).

## 6. TimelineEvent (abstract) and subtypes

Abstract generalization (△ in the LLD class diagram) over all clinical happenings. Concrete subtypes shown in the diagram: **Diagnosis, Prescription, Hospitalization, Report**. The prototype's event stream additionally uses **Consultation, Vaccination, Document** event types with the same base shape; they are specified here as further subtypes.

### 6.1 Base attributes (all subtypes inherit)

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `event_id` | UUID | ✔ | |
| `patient_id` | FK → Patient | ✔ | Global reference. |
| `tenant_id` | FK → TenantOrganization | ✔ | Producing organization — ownership anchor. |
| `type` | enum | ✔ | See taxonomy §7. |
| `occurred_at` | timestamptz | ✔ | Original clinical timestamp — from source provenance for synchronized records, not import time (PRD §23). Hospitalization uses a range (see §6.5). |
| `provider` | string / FK | ✔ | Authoring clinician ("Dr. R. Menon"); FK → Practitioner when Atlas-originated, verbatim string from provenance when imported (Proposed split). |
| `organization` | string | ✔ | Display organization ("City General", "Apollo Diagnostics"). |
| `provenance` | Provenance | ✔ | Embedded value object, §10. |
| `version` | int | ✔ | Increments on amendment; amendments create new versions, never overwrite (Architecture §11.2). |
| `attachments` | Document 0..* | – | Proposed generalization: PRD §5 lists attachments as event metadata; the class diagram draws the association from Report (§9). |
| `notes` | text | – | Clinical notes (PRD §5). |
| `related_event_ids` | UUID[] | – | Proposed: PRD §5 "related prescriptions / related reports" linkage. |

### 6.2 Consultation (`CON`)

Not detailed in the class diagram; attribute set is Proposed, grounded in PRD §5 metadata and the prototype's consult flow.

| Attribute | Type | Notes |
|-----------|------|-------|
| `reason` | string | "Follow-up — Type 2 diabetes". |
| `notes` | text | Findings/plan/advice; supports voice dictation input (doctor app). |
| `related_diagnosis_ids`, `related_prescription_ids` | UUID[] | Prescriptions written and diagnoses recorded in the consult. |

### 6.3 Diagnosis (`DX`)

| Attribute | Type | Notes |
|-----------|------|-------|
| `code` | string (ICD-10) | LLD `code: ICD-10`; optional at capture ("Code (optional) — ICD-10, e.g. H10.1"); architecture allows standardized terminology to be introduced later (PRD §7). |
| `status` | enum | LLD; Proposed values `active | resolved | ruled_out`. |
| `notes` | text | |
| `diagnosed_by` | FK/string | LLD; PRD §7 diagnosing doctor + organization. |

Example instance: "Type 2 diabetes (E11.9)" · Dr. S. Iyer · 14 Mar 2022.

### 6.4 Prescription (`RX`)

| Attribute | Type | Notes |
|-----------|------|-------|
| `drug` | string | "Metformin 500 mg". |
| `dose` | string | LLD `dose`. |
| `route` | string | LLD `route` (PRD §8). |
| `frequency` | string | "Twice daily · with meals". |
| `duration` | string | "once daily · 14 days" pattern in capture form. |
| `instructions` | text | "With food, avoid alcohol…" (PRD §8). |
| `status` | enum | LLD shows `active | done`; PRD §8 requires distinguishing **active, completed, discontinued, historical** — model the full set, map `done → completed` for the prototype's "Done" pill. |
| `prescribed_by`, `prescribed_at`, `start/end date` | | PRD §8. |

Writing a prescription that conflicts with a recorded allergy (penicillin-class demo) requires an override reason and flags the save to compliance — the override reason and flag live on the audit trail, not the prescription itself (prototype toast "Saved with allergy override — flagged to compliance"; Proposed persistence detail).

### 6.5 Hospitalization (`HSP`)

| Attribute | Type | Notes |
|-----------|------|-------|
| `admitted_at` / `discharged_at` | timestamptz | LLD; displayed as a range "09 – 12 Nov 2025". |
| `reason` | string | "Acute appendicitis" (PRD §9 reason for admission). |
| `procedures` | string[] | LLD `procedures[]`; "Laparoscopic appendectomy". |
| `discharge_summary` | text / Document ref | LLD; the summary PDF is an attachment. |
| `doctors_involved` | string[] | PRD §9; "Dr. A. Fernandes (Surgery)". |
| `discharge_condition` | string | "Stable, recovered". |
| `meds_at_discharge` | string[] | "Amoxicillin 500 mg · 5 days". |
| `follow_up_instructions` | text | Fixed copy example in prototype: "Wound review in 10 days. Avoid heavy lifting for 4 weeks. …". |
| `diagnoses`, `treatments`, `investigations` | | PRD §9 remaining metadata (Proposed as linked events / structured lists). |

### 6.6 Report (`LAB`)

Laboratory, imaging, and other diagnostic reports. Carries the publication lifecycle — see §8.

| Attribute | Type | Notes |
|-----------|------|-------|
| `status` | enum | Lifecycle enum (LLD `status: lifecycle enum`), §8. |
| `verified_at` | timestamptz | LLD; provenance block shows "Verified 21 Aug 2026, 9:02". |
| `published_at` | timestamptz | LLD `published_at`. |
| `amended_of` | FK → Report? | LLD `amended_of?: FK` — points at the report this one amends. |
| `name` | string | "Lipid panel", "HbA1c", "Chest X-ray", "Complete blood count", "Thyroid panel (TSH, T4)". |
| `results` | analyte rows | Structured values with units and flags ("Triglycerides 168 mg/dL ▲"); enables trend charts and side-by-side compare (Proposed shape: `{analyte, value, unit, flag}`; grounded in report-detail and compare screens). |

### 6.7 Vaccination (`VAX`)

Prototype instance: "Influenza vaccine" · City General · 18 Feb 2026. Attributes Proposed beyond the base: `vaccine` (string), `dose_number`, `lot_number`.

### 6.8 Document event (`DOC`)

A standalone document upload as a timeline entry (doctor "Add to record → Report" flow produces `chip DOC`, e.g. "Clinical photo — left eye"). Carries a `title` plus one Document attachment (§9).

## 7. Event taxonomy — PRD §5 types → timeline chips

Chips are 38×38 px rounded squares; for hue `h`: background `oklch(0.93 0.045 h)`, label text `oklch(0.4 0.12 h)` (prototype `decorate()`; tokens in [01-design-system.md](01-design-system.md)).

| PRD §5 event type | Subtype (entity) | Chip | Hue `h` | Chip bg / text | In prototype |
|---|---|---|---|---|---|
| Doctor consultation | Consultation | `CON` | 280 | `oklch(0.93 0.045 280)` / `oklch(0.4 0.12 280)` | ✔ |
| Diagnosis | Diagnosis | `DX` | 25 | `oklch(0.93 0.045 25)` / `oklch(0.4 0.12 25)` | ✔ |
| Prescription | Prescription | `RX` | 150 | `oklch(0.93 0.045 150)` / `oklch(0.4 0.12 150)` | ✔ |
| Laboratory test | Report | `LAB` | 235 | `oklch(0.93 0.045 235)` / `oklch(0.4 0.12 235)` | ✔ |
| Imaging/report | Report | `LAB` | 235 | same | ✔ (Chest X-ray) |
| Hospital admission / discharge | Hospitalization (one event spanning both) | `HSP` | 60 | `oklch(0.93 0.045 60)` / `oklch(0.4 0.12 60)` | ✔ |
| Vaccination | Vaccination | `VAX` | 190 | `oklch(0.93 0.045 190)` / `oklch(0.4 0.12 190)` | ✔ |
| Other clinical documents | Document event | `DOC` | 235 | `oklch(0.93 0.045 235)` / `oklch(0.4 0.12 235)` | ✔ |
| Procedure / Surgery | Recorded within Hospitalization `procedures[]`; standalone procedure events Proposed: reuse `HSP` chip | `HSP` | 60 | — | ✔ (within HSP) |
| Referral | Proposed subtype, chip `REF`, hue TBD by design | — | — | — | ✘ |
| Follow-up | Not a stored event — surfaced as a notification with Confirm/Reschedule (see [02-patient-app.md](02-patient-app.md)); Proposed: persists as a scheduled item, not a TimelineEvent | — | — | — | ✔ (as notification) |

Every timeline card shows `{date} · {type label}` (e.g. "Today · Lab report"), a bold title, the organization line, and — when the event has at least one attachment — a `PDF` badge (`oklch(0.94 0.02 235)` bg / `oklch(0.42 0.12 235)` text).

**Acceptance criteria**

- Each stored event maps to exactly one chip code; unknown/other types must not render an empty chip (Proposed fallback: `DOC`).
- Chip colors derive from the single hue formula — adding a type is a taxonomy row, not new CSS.

## 8. Report lifecycle and publication (PRD §24)

### 8.1 State machine

```text
ORDERED → IN_PROGRESS → COMPLETED → VERIFIED → FINAL → PUBLISHED_TO_PATIENT
```

- Integration ingest of a finalized report enters at `FINAL` ("stored v1 · status FINAL", LLD sequence B) and then evaluates the tenant's publication policy.
- Publication policy outcomes per report type (TenantOrganization.`publication_rules`): auto-publish after finalization, publish after verification, **hold for manual release**, or **restricted to healthcare professionals** (clinician-only visibility — LLD activity diagram `[hold]` branch).
- Only reports meeting the configured publication criteria become visible to patients (PRD §24). Doctors with consent can see clinician-only reports.
- On `PUBLISHED_TO_PATIENT`, the platform emits `report.finalized` → PHI-free push "New report available" (LLD sequence B; Architecture §12).

### 8.2 Amendment

If a report is corrected after publication: a **new Report version** is created with `amended_of` pointing at the prior report, provenance retained on both, and the record visibly indicates that an updated version exists (PRD §24; Architecture §11.2). Embeddings for the record are regenerated (Architecture §10.4).

### 8.3 Patient/doctor-facing status pills

The UI compresses the machine states into three display statuses (prototype `statusPill`):

| Display status | Meaning | Pill bg / text |
|---|---|---|
| `Final` | `FINAL` or `PUBLISHED_TO_PATIENT` | `oklch(0.93 0.05 150)` / `oklch(0.42 0.12 150)` |
| `Preliminary` | Pre-final states visible to clinicians (`COMPLETED`/`VERIFIED`) | `oklch(0.93 0.06 80)` / `oklch(0.45 0.12 60)` |
| `Amended` | Superseded — a newer version exists (`amended_of` chain) | `oklch(0.94 0.02 235)` / `oklch(0.42 0.12 235)` |

Report detail header renders the status inline in green for Final ("Apollo Diagnostics · 21 Aug 2026 · **Final**", color `oklch(0.5 0.12 150)`).

**Acceptance criteria**

- A `Preliminary` report is never shown in the patient app unless the tenant's policy publishes pre-verification states (default: not shown; the prototype shows "Thyroid panel — Preliminary" in the patient list, so publication of preliminaries is a per-tenant choice, not a hard block).
- Amending a `PUBLISHED_TO_PATIENT` report never deletes or mutates the original row; both versions remain retrievable with distinct `version` numbers.
- State transitions are forward-only; any skip (e.g. ingest at FINAL) is recorded via provenance `record_status`, not by back-filling intermediate states (Proposed).

## 9. Document (attachment)

Binary artifacts: reports, discharge summaries, referral letters, clinical photos, DICOM studies (PRD §6 type list).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `doc_id` | UUID | ✔ | |
| `mime` | string | ✔ | Supported at upload: PDF, JPG, DICOM (prototype upload card: "PDF, JPG, DICOM · scanned for malware"); PRD §6 requires common document/image formats generally. |
| `size` | int | ✔ | Large files must not block the app (PRD §31 — pre-signed URL up/download, Architecture §9). |
| `scan_status` | enum | ✔ | `clean | blocked` (LLD). A document is invisible everywhere until the malware scan passes; `blocked` documents are never served (Architecture §9 "malware-scanning pipeline before records become visible"). Proposed intermediate: `pending` while the scan runs. |
| `storage_ref` | string | ✔ | `s3://…` object key in the tenant's bucket/prefix (LLD; Architecture §13). |
| `title` | string | – | Capture-form title ("Clinical photo — left eye"). |
| `metadata` | object | – | PRD §6 requires document metadata (uploaded_by, uploaded_at, linked event). |

Associations: Report `1 — 0..*` Document (class diagram). Proposed generalization: any TimelineEvent may carry attachments (PRD §6: "associated with the relevant patient and medical event"; discharge summaries and consult photos need it).

Rules:

- **Immutability:** stored content is never edited in place; a corrected document is a new Document (new `doc_id`) attached to the amending record version (Proposed, consistent with §8.2 versioning). Object storage applies lifecycle rules and legal hold (Architecture §13).
- Access is authorization-controlled and every view/download is audited (PRD §6, §13).
- Downloads/uploads use short-lived pre-signed URLs; documents are never proxied through the API tier (Architecture §9).

**Acceptance criteria**

- An uploaded file is retrievable by no role (including the uploader) until `scan_status = clean`.
- A `blocked` document produces an audit event and an uploader-facing error, and is never listed in patient or doctor apps (Proposed error surface).

## 10. Provenance (value object, PRD §27)

Embedded on every TimelineEvent (`provenance: Provenance`, LLD). Populated by the integration pipeline for synchronized records and by the writing service for Atlas-originated records; **kept verbatim, never overwritten** by mapping/normalization (LLD activity node "Persist + provenance — source system, record id, timestamps, author, version — kept verbatim").

| Field | Type | Example (prototype provenance block / PRD §27) |
|-------|------|--------------------------------|
| `source_system` | string | `Apollo LIS` / `Hospital EHR` |
| `source_organization` | string | `Example Hospital` |
| `source_record_id` | string | `LAB-928372` |
| `source_timestamp` | timestamptz | `2026-08-21T10:30:00Z` — original clinical timestamp |
| `verified_at` | timestamptz | "Verified 21 Aug 2026, 9:02" (report provenance card) |
| `imported_at` | timestamptz | "Imported 9:03" / `2026-08-21T10:31:02Z` |
| `record_status` | string | `FINAL` — status as declared by the source |
| `author` | string | Author/provider from the source (PRD §23) |
| `version` | int/string | Source version where applicable |
| `origin` | enum | Proposed: `atlas | integration` — PRD §23 requires distinguishing Atlas-originated vs synchronized records; Atlas-originated records set `source_system = "Atlas"` with the authoring practitioner. |

UI contract: the patient report detail renders a provenance card with exactly this content —

```text
Provenance
Source: Apollo LIS · LAB-928372
Verified 21 Aug 2026, 9:02 · Imported 9:03
Status: FINAL · published to patient
```

**Acceptance criteria**

- Re-processing, normalization, or amendment of a record leaves all provenance fields of existing versions byte-identical.
- Every synchronized record answers: which system, which org, which source ID, when it happened, when Atlas imported it, and what status the source declared (PRD §27's stated purpose: auditability, clinical trust, reconciliation, interoperability).

## 11. ConsentGrant

First-class consent object, evaluated by the policy engine on every read (Architecture §2, §7). Full behavioral spec in [07-consent-and-access-control.md](07-consent-and-access-control.md).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `grant_id` | UUID | ✔ | |
| `patient_id` | FK → Patient | ✔ | Grantor. |
| `grantee_id` | FK → Practitioner (or TenantOrganization) | ✔ | LLD cardinality "1 grantee"; org-level grantees per PRD §12 (e.g. Apollo Diagnostics "Integration · sends reports only"). |
| `categories` | enum[] | ✔ | «mh default-off» (LLD). Category set (Architecture §7): `general_history`, `reports_labs`, `medications`, `diagnoses`, `mental_health` (**default off**), `sensitive`. Consent UI toggles: General history, Reports & labs, Medications, Mental health. |
| `duration` | enum | ✔ | Picker values: `24 hours | 30 days | 90 days`. QR-scan consult grants default to 30 days (`consultGrant(30d, categories)`, LLD sequence A). |
| `purpose` | string | ✔ | Records the grant path: `consult | requested | org-policy | integration | emergency` ([07-consent-and-access-control.md](07-consent-and-access-control.md) §3.1); distinct from the per-request purpose-of-use ABAC input (07 §2.2). |
| `status` | enum | ✔ | `requested | active | denied | expired | revoked` — the LLD shows only the last three; `requested`/`denied` extend it per the grant lifecycle in [07-consent-and-access-control.md](07-consent-and-access-control.md) §3.1/§3.3. |
| `expires_at` | timestamptz | ✔ | All grants auto-expire ("Expires 12 Sep 2026"). |
| `is_emergency` | boolean | ✔ | LLD. Break-glass grants: constrained to emergency profile + critical history, compliance + patient alerted, flagged for review (Architecture §7). |

Creation paths (Architecture §7): (a) QR scan → implicit consult grant with default duration; (b) explicit patient approval of a request (duration + category picker); (c) organization policy for records the org itself created. Revocation takes effect immediately — AI retrieval and 360° composition are authorization-filtered at read time (Architecture §10.2).

State machine (full five-state machine in [07-consent-and-access-control.md](07-consent-and-access-control.md) §3.3): `requested → active` (patient approves) or `requested → denied`; `active → expired` (clock) and `active → revoked` (patient action); a revoked grant can be re-granted only as a **new** grant (prototype "Re-grant" action on the revoked Dr. A. Fernandes row; Proposed as new-row semantics for audit integrity).

Status pills: Active `oklch(0.93 0.05 150)`/`oklch(0.42 0.12 150)`; Revoked `oklch(0.94 0.04 25)`/`oklch(0.45 0.15 25)`.

## 12. IdentityMapping (IdentityLink)

Per-tenant crosswalk from an external hospital identifier to the global Atlas identity (PRD §22).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `tenant_id` | FK → TenantOrganization | ✔ | |
| `external_id` | string | ✔ | e.g. `HOSP-928372`. |
| `atlas_id` | FK → Patient | ✔ | e.g. `ATL-82X92K`. |
| `confidence` | float | ✔ | Probabilistic match score; sequence B shows `resolve(HOSP-928372) → ATL-82X92K (confidence 0.98)`. |
| `status` | enum | ✔ | `matched | pending` (LLD). `pending` = ambiguous match sitting in the human resolution queue. |

Rules:

- Ambiguous matches route to explicit human resolution — **never auto-attach clinical data to the wrong patient** (PRD §22; LLD activity «constraint»; admin queue action "Resolve identity" → "Routed to identity resolution — never auto-attached").
- Every identity-linking operation is audited (PRD §22).
- Unique per `{tenant_id, external_id}`; one patient may hold many mappings (one or more per integrated tenant).

ABHA linkage is separate: the patient's `abha_address` on the Patient record (§3) covers ABDM federation; IdentityMapping covers hospital-system IDs.

## 13. AuditEvent

Append-only, hash-chained record of every sensitive operation (PRD §13). **References, never owns, clinical entities** (LLD).

| Attribute | Type | Req | Notes |
|-----------|------|-----|-------|
| `audit_id` | UUID | ✔ | |
| `actor_id` | string | ✔ | User or system client. |
| `role` | enum | ✔ | Patient / doctor / admin / system (PRD §13 metadata). |
| `tenant_id` | FK | ✔ | Acting tenant context. |
| `patient_id` | FK | ✔ | Subject patient (reference only). |
| `action` | enum | ✔ | PRD §13 event list: record viewed, document viewed/downloaded/uploaded, QR access, access granted/revoked, record created/modified, diagnosis added, prescription added; plus break-glass, AI generation, identity link (Architecture §9, §10.5; PRD §22). |
| `resource` | string | ✔ | Resource type + id acted on. |
| `access_method` | enum | ✔ | `qr | search | api | integration | break_glass | emergency_card` (PRD §13 "access method"; values per [07-consent-and-access-control.md](07-consent-and-access-control.md) §8.2 — `break_glass` and `emergency_card` are distinct so a doctor break-glass session and a lock-screen card open remain distinguishable, 07 §8.3). |
| `result` | enum | ✔ | e.g. `permit | deny` (policy decisions are logged with inputs, Architecture §7). |
| `ts` | timestamptz | ✔ | |
| `session_ref` | string | – | Proposed (not in the LLD class box): request/session context — PRD §13 metadata includes "relevant request/session information"; QR resolution binds the token to the doctor's session (`sessionBind`, LLD sequence A) and the doctor UI shows "Session 28:44 · audited". |
| `hash_prev` | sha256 | ✔ | Hash chain for tamper evidence (PRD §13 "tamper-resistant"); stored in WORM with immutable retention, anchored daily (Architecture §13). |

Patient-facing projection: the "Recent access log" card renders audit rows as `"{date time} — {actor} · {action}"` (e.g. "12 Aug 10:41 — Dr. R. Menon · viewed via QR").

**Acceptance criteria**

- Deleting or updating an AuditEvent is impossible through any API; chain verification detects gaps.
- Every QR scan produces at least three audit events: scan, grant, view (LLD sequence A message 13).

## 14. IntegrationEvent

One inbound message from a tenant connector traversing the ingest pipeline (see [11-integrations.md](11-integrations.md) for the pipeline itself). Not in the class diagram; shape is Proposed, states and surfaced fields are grounded in PRD §25–26, Architecture §11.2, and the admin error-queue screen.

| Attribute | Type | Notes |
|-----------|------|-------|
| `event_id` | UUID | Returned to the source in the `202 Accepted {event_id}` ack (LLD sequence B). |
| `tenant_id` | FK | Per-tenant queues and DLQs (Canvas invariant 5). |
| `connector` | string | "Apollo LIS", "Radiology PACS", "Legacy HIS". |
| `resource_type` | string | Lab result, discharge summary, imaging report… |
| `source_record_id` | string | "LAB-102993", "DS-5521", "IMG-8817". |
| `idempotency_key` | tuple | `{tenant, source_system, source_record_id, version}` — duplicates acknowledged (200) and discarded, safe replay (Architecture §11.2; LLD activity). |
| `state` | enum | See below. |
| `failure_reason` | string | PHI-minimized ("HOSP-928372 → ambiguous identity (2 candidate matches)", "Schema validation failed · missing encounter reference", "Endpoint timeout"). |
| `attempt` / `max_attempts` / `next_retry_at` | int/int/ts | "attempt 3 of 5 · next retry 11:20"; exponential backoff (Architecture §11.2). |
| `payload_ref` | string | Raw payload held for the secure viewer — admins see metadata only ("Payload opened in secure viewer (metadata only)"). |

### 14.1 Queue states (PRD §26)

```text
RECEIVED → VALIDATING → PROCESSED
                      → REJECTED
                      → RETRYING → PROCESSED | FAILED → DEAD_LETTER
```

| State | Meaning | Admin UI |
|-------|---------|----------|
| `RECEIVED` | Accepted and enqueued (source got 202). | — (counts only) |
| `VALIDATING` | Schema/auth checks running. | — |
| `PROCESSED` | Persisted with provenance; terminal success. | Health line ("1,204 events today · healthy") |
| `REJECTED` | Refused deterministically (e.g. unauthenticated source → 401, alert tenant admin). | — |
| `RETRYING` | Transient failure, backoff in progress. | Amber pill — bg `oklch(0.93 0.06 80)`, text `oklch(0.45 0.12 60)` |
| `FAILED` | Retries exhausted or needs human action (e.g. ambiguous identity). | Red pill — bg `oklch(0.94 0.04 25)`, text `oklch(0.45 0.15 25)`; offers **Resolve identity** where applicable |
| `DEAD_LETTER` | Permanently unprocessable (e.g. schema failure); parked, never dropped. | Neutral pill — bg `rgba(34,48,60,0.1)`, text `rgba(34,48,60,0.65)`; offers **View payload** |

Queue-level action: **Retry all recoverable** re-enqueues `RETRYING`/`FAILED` (not `DEAD_LETTER`) events.

**Acceptance criteria**

- No inbound clinical message can reach a state outside this machine; every failure path terminates in a state visible to tenant admins (LLD activity: "Every failure path terminates in a visible state; clinical data is never silently dropped").
- Replaying an already-`PROCESSED` idempotency key returns success to the source without creating a second TimelineEvent.
- `FAILED` events with ambiguous identity link to the resolution queue and create `IdentityMapping(status: pending)` rows, never a TimelineEvent.

## 15. FamilyLink

Family profiles under one login: the patient home shows a member switcher (Priya · Aarav · Kamala Sharma) and greets the selected member. Grounded in Patient.`guardian_links: Link[]` (LLD) and Architecture §4 "family profiles (guardian links)"; attribute shape Proposed.

| Attribute | Type | Notes |
|-----------|------|-------|
| `link_id` | UUID | |
| `guardian_atlas_id` | FK → Patient | The account holder (Priya). |
| `member_atlas_id` | FK → Patient | The managed profile (Aarav, Kamala) — each member is a full global Patient with their own `atlas_id`, QR, and timeline. |
| `relationship` | enum | Proposed: `child | parent | spouse | other`. |
| `status` | enum | Proposed: `active | revoked`. |

Rules: switching members switches the entire data context (greeting name, QR, timeline); a guardian's access to a member's records is itself an authorization edge evaluated by the policy engine and audited (Proposed, consistent with "everything sensitive is audited").

## 16. Relationship catalog (ER view)

Cardinalities as drawn in the LLD class diagram unless marked otherwise.

```text
Patient            1 ── *    ConsentGrant        (patient grants)
Practitioner       1 ── *    ConsentGrant        ("1 grantee")
Patient            1 ── *    TimelineEvent       (clinical stream)
Patient            1 ── *    IdentityMapping     (external-ID crosswalk)
TenantOrganization 1 ── *    IdentityMapping
TenantOrganization 1 ── *    Practitioner        ("employs")
TenantOrganization 1 ── *    TimelineEvent       (tenant_id FK on base; implied)
TimelineEvent      △──       Diagnosis | Prescription | Hospitalization | Report
                             (generalization; extended: Consultation, Vaccination,
                              Document event — prototype event types)
Report             1 ── 0..* Document            (attachments)
AuditEvent         ┄┄▷       Patient             («references» — never owns)
Patient            * ── *    Patient  via FamilyLink (guardian_links; Proposed shape)
TimelineEvent      1 ── 1    Provenance          (embedded value object)
Report             0..1 ──   Report  (amended_of self-reference)
```

## 17. Timeline grouping, sorting, and filtering rules

Applies identically to the patient Timeline screen and the doctor 360° Timeline tab (they render the same stream through the same rules).

1. **Sort:** events ordered by `occurred_at`, newest first. A hospitalization sorts by its admission date and displays its full range ("09–12 Nov").
2. **Group:** consecutive events group by **calendar year** of `occurred_at`. Group order follows the sort (newest year first: 2026, 2025, 2022). Empty years are skipped — no placeholder groups.
3. **Group header:** the year as a small tracked label ("2026") with a hairline rule; date within a card omits the year ("12 Aug", "Today" for the current day).
4. **Filters (patient timeline):** All · Consultations · Reports · Hospital. Filter → type mapping (prototype `tlMatch`):
   - Consultations → `Consultation`, `Diagnosis`
   - Reports → `Report` (Lab report), `Document`
   - Hospital → `Hospitalization`
   Filtering re-groups: years with no matching events disappear.
5. **Subtitle:** unfiltered — `"{count} events · {oldest year} – today · from {n} organizations"` (e.g. "7 events · 2022 – today · from 3 organizations"); filtered — `"{count} event(s) · {filter noun}"` with correct singular/plural.
6. **New events prepend:** a record saved during a consult appears immediately at the top of the current year's group (prototype `dSave` behavior).
7. **Tap targets:** events with a viewable artifact (report with document) navigate to their detail screen; others are inert cards (prototype: only the lipid-panel event is tappable — production rule Proposed: every event with a detail screen is tappable).

**Acceptance criteria**

- Given the seven persona events, the timeline renders exactly three groups — 2026 (5 events: lipid panel, consultation, Metformin renewal, HbA1c, influenza vaccine), 2025 (appendectomy), 2022 (T2DM diagnosis) — in that order.
- Selecting "Reports" yields 2 events (lipid panel, HbA1c — a DOC upload saved during a consult would join them) and drops the 2025 and 2022 groups entirely (prototype `tlMatch`: both remaining events are 2026).
- Grouping and filter logic live in one shared implementation consumed by both apps.

## 18. Sample persona dataset (canonical test fixture)

Implementations should seed this dataset (from the prototype) for demos and integration tests:

- **Patient:** Priya Sharma, F, DOB 14 May 1992, `ATL-82X92K`, blood group B+, allergy Penicillin (severe, 2019), conditions Type 2 diabetes (E11.9, 2022) + allergic rhinitis, emergency contact Aman S. +91 98111 20844, ABHA `priya@abdm` (linked), family Aarav Sharma and Kamala Sharma.
- **Practitioners:** Dr. Rohan Menon (Endocrinology, City General Hospital), Dr. Meera Kulkarni (Dermatology, SkinCare Clinic), Dr. A. Fernandes (Surgery, City General), Dr. S. Iyer.
- **Timeline events:** the seven events listed in §17 acceptance criteria, with chips/hues per §7.
- **Reports:** Lipid panel (Final, 21 Aug 2026, provenance `Apollo LIS · LAB-928372`), Thyroid panel (Preliminary), HbA1c 6.8% (Final, 03 Jun 2026), Chest X-ray (Amended, 10 Nov 2025), CBC (Final, 09 Nov 2025).
- **Prescriptions:** Metformin 500 mg BD (Active), Atorvastatin 10 mg OD night (Active), Amoxicillin 500 mg (Completed, post-surgery Nov 2025).
- **Consents:** Dr. R. Menon (Active, expires 12 Sep 2026, from QR scan 12 Aug), Apollo Diagnostics (Active, ongoing, integration send-only), Dr. A. Fernandes (Revoked 20 Nov 2025), pending request from Dr. Meera Kulkarni.
- **IntegrationEvents (Legacy HIS):** LAB-102993 `FAILED` (ambiguous identity, 2 candidates), DS-5521 `DEAD_LETTER` (schema: missing encounter reference), IMG-8817 `RETRYING` (timeout, attempt 3 of 5).
