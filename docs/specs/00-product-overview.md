# Product overview

Defines what Atlas is, who it serves, what ships in the MVP, and the vocabulary every other spec in this set uses.

Sources: `design/uploads/Atlas_Project_Requirements.md` (PRD, all 35 sections; primarily §§1–3, 30, 33–35), `design/Atlas App.dc.html` (built prototype — 21 screens across three roles), `design/Canvas.dc.html` (tenancy model and invariants), `design/Atlas Architecture.dc.html` (glossary grounding), `design/Atlas Directions.dc.html` (chosen visual direction), `design/ios-frame.jsx` / `design/android-frame.jsx` (platform targets).

---

## 1. Vision and core concept

Atlas is a secure, patient-centric digital health record platform. A patient's medical information is stored in one place and made available to authenticated healthcare professionals through a controlled QR-based access mechanism (PRD §1).

The core concept, verbatim from the PRD:

> **One patient → One identity → One QR → Complete medical history → Controlled doctor access**

Atlas gives doctors a consolidated view of relevant patient information while ensuring every access is authenticated, authorized, auditable, and privacy-conscious.

Three settled platform decisions frame everything else in this spec set:

1. **Multi-tenant SaaS.** A tenant is a healthcare organization (hospital, clinic chain). See [10-architecture.md](10-architecture.md).
2. **Global identity, tenant-owned records.** Patient identity is platform-global — patients are platform-level identities, not tenant members. Clinical records are owned by the tenant that created them. The patient's 360-degree view is a **consented read-time composition** across tenants; no tenant's data is copied into another tenant's store. See [09-data-model.md](09-data-model.md) and [07-consent-and-access-control.md](07-consent-and-access-control.md).
3. **AI summaries are a top-priority requirement** (added after the PRD): a plain-language health summary on the patient home, an AI clinical brief generated at QR-scan time on the doctor side, ask-the-record follow-up Q&A, and per-report plain-language explanation — all with strict disclaimers. See [05-ai-features.md](05-ai-features.md).

### Acceptance criteria

- Every feature in this spec set traces to the PRD, the approved enhancement list (§4 below), or the prototype; anything else is labeled "Proposed:".
- No design in any sibling spec allows a tenant to read another tenant's clinical records except through the consented read-time composition path or break-glass.

---

## 2. Goals

From PRD §2. These are the product-level goals every spec must serve:

| # | Goal |
|---|------|
| G1 | Maintain a centralized digital record for each patient. |
| G2 | Make a patient's medical history available across healthcare encounters. |
| G3 | Allow authenticated doctors to quickly access patient information using a QR code. |
| G4 | Give doctors a consolidated view of the patient's medical journey. |
| G5 | Support structured medical information as well as uploaded reports/documents. |
| G6 | Maintain a complete audit trail of access and changes. |
| G7 | Ensure patients retain control over who can access their information. |
| G8 | Design the platform so it can evolve into a broader healthcare interoperability platform. |

---

## 3. Personas and capabilities

Three roles (PRD §3). The prototype's role switcher (Patient / Doctor / Admin) mirrors this split, and the app specs are organized the same way.

### 3.1 Patient

Owns or is associated with a digital Atlas profile containing their medical history. Sample persona used across the prototype and all spec examples: **Priya Sharma**, 34 F, Atlas ID `ATL-82X92K`, blood group B+, severe penicillin allergy (2019), Type 2 diabetes (dx 2022, E11.9), Metformin 500 mg BD + Atorvastatin 10 mg OD, appendectomy Nov 2025, ABHA `priya@abdm`; family members Aarav and Kamala Sharma.

Patients can (PRD §3.1):

- Register / create their profile.
- View their medical information, reports and documents, diagnoses and treatment history, prescriptions and discharge summaries.
- Generate / display their Atlas QR code.
- See which doctors or healthcare organizations have accessed their records.
- Grant or revoke access where applicable.
- Manage basic profile information.

Post-PRD additions for this role (all in the prototype; see §4): family profiles under one login, offline emergency card, consent request approval with duration and per-category control, AI plain-language summary, follow-up confirm/reschedule, the HbA1c trend chart, multi-language UI, biometric/passkey sign-in. Full screen-by-screen spec: [02-patient-app.md](02-patient-app.md).

### 3.2 Doctor / healthcare professional

Uses Atlas to retrieve patient information during a consultation. Sample personas: **Dr. Rohan Menon**, Endocrinology, City General Hospital (primary); **Dr. Meera Kulkarni**, Dermatology, SkinCare Clinic (second consent requester).

Doctors can (PRD §3.2):

- Authenticate securely.
- Scan a patient's Atlas QR code.
- Request/access patient information according to authorization rules.
- View a consolidated patient profile (the 360-degree view).
- Review previous diagnoses, consultations, reports, prescriptions, procedures, and discharge summaries.
- Add new clinical information; record consultations and diagnoses.
- Upload medical documents/reports.
- View relevant historical information before making clinical decisions.

Post-PRD additions: patient search by name / Atlas ID / phone, emergency break-glass access, AI clinical brief at scan time with ask-the-record, allergy conflict warning when prescribing, voice dictation for clinical notes, side-by-side report compare. Full spec: [03-doctor-app.md](03-doctor-app.md).

### 3.3 Healthcare organization / Admin

The PRD (§3.3) framed this role as optional ("may be introduced"); the approved gap analysis made it firm and it is built in the prototype (Admin dashboard + integration error queue). Admins manage:

- Doctors and staff.
- Organization profiles.
- Access policies.
- Patient records created by the organization.
- Audit logs.
- Organization-level configuration.
- Integration configuration, monitoring, and the integration error queue (PRD §§21, 26; enhancement #5).

Full spec: [04-admin-app.md](04-admin-app.md); integration operations detail: [11-integrations.md](11-integrations.md).

### Acceptance criteria

- Each role sees only the app surface specified for it; there is no cross-role navigation inside the product (the prototype's role switcher is a demo affordance, not a product feature).
- Every capability bullet above is realized by at least one screen or API in the sibling specs.

---

## 4. Approved post-PRD enhancements

Eighteen gap-analysis recommendations were approved after the PRD was written. All 18 are present in the built prototype and are in scope for the specs. "Where it lives" names the prototype screen(s) (the `data-screen-label` values in `Atlas App.dc.html`).

| # | Feature | Role | Where it lives in the app | Covering spec |
|---|---------|------|---------------------------|---------------|
| 1 | Doctor patient search (name / Atlas ID / phone) | Doctor | Doctor home | [03-doctor-app.md](03-doctor-app.md) |
| 2 | Consent grant flow: duration picker (24 hours / 30 days / 90 days) + per-category toggles, mental health off by default | Patient | Consent request | [02-patient-app.md](02-patient-app.md), [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| 3 | Report lifecycle states (Final / Preliminary / Amended) | Patient, Doctor | Patient records, Report detail | [09-data-model.md](09-data-model.md), [11-integrations.md](11-integrations.md) |
| 4 | Hospitalization / discharge detail screen | Patient | Discharge detail | [02-patient-app.md](02-patient-app.md) |
| 5 | Admin org role with integration error queue | Admin | Admin dashboard, Admin integration queue | [04-admin-app.md](04-admin-app.md), [11-integrations.md](11-integrations.md) |
| 6 | Emergency break-glass access | Doctor | Emergency break-glass access | [03-doctor-app.md](03-doctor-app.md), [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| 7 | Family profiles under one login (Aarav, Kamala Sharma) | Patient | Patient home (profile switcher) | [02-patient-app.md](02-patient-app.md), [06-identity-auth.md](06-identity-auth.md) |
| 8 | Consent granularity (category-level rules; sensitive categories gated) | Patient | Consent request, Access and consent | [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| 9 | Biometric login + passkey ("Sign in with Face ID") | Patient | Patient welcome, Patient profile ("Face ID + passkey" under Security & sign-in) | [06-identity-auth.md](06-identity-auth.md) |
| 10 | Offline lock-screen emergency card | Patient | Emergency card | [02-patient-app.md](02-patient-app.md) |
| 11 | HbA1c / vitals trend chart | Patient | Report detail ("TREND — HBA1C" card) | [02-patient-app.md](02-patient-app.md) |
| 12 | Follow-up confirm / reschedule | Patient | Notifications (reminder with Confirm / Reschedule) | [02-patient-app.md](02-patient-app.md) |
| 13 | Multi-language UI: English / Hindi / Marathi | Patient | Patient welcome (language chips), app-wide | [02-patient-app.md](02-patient-app.md), [13-non-functional.md](13-non-functional.md) |
| 14 | Allergy conflict warning on prescribing (penicillin-class demo) | Doctor | Doctor add to record | [03-doctor-app.md](03-doctor-app.md) |
| 15 | Ask-the-record AI (follow-up Q&A chips under the AI clinical brief) | Doctor | Doctor patient 360 | [05-ai-features.md](05-ai-features.md) |
| 16 | Voice dictation for clinical notes | Doctor | Doctor add to record | [03-doctor-app.md](03-doctor-app.md) |
| 17 | Side-by-side report compare | Doctor | Doctor patient 360 (Reports tab, List / Compare toggle) | [03-doctor-app.md](03-doctor-app.md) |
| 18 | ABDM / ABHA alignment (India-first) — "Works with ABHA (ABDM)", ABHA address on profile | Platform | Patient welcome, Patient profile | [06-identity-auth.md](06-identity-auth.md), [12-api-and-interoperability.md](12-api-and-interoperability.md) |

The AI summary features (patient plain-language summary, doctor clinical brief at scan, ask-the-record — #15 plus the summary surfaces) were elevated to **top priority** after the PRD; [05-ai-features.md](05-ai-features.md) is their authoritative spec. In the prototype the ask-the-record chips appear only under the doctor's AI clinical brief; whether ask-the-record is also offered on the patient summary is settled in [05-ai-features.md](05-ai-features.md).

### Acceptance criteria

- All 18 enhancements are specified in the named sibling spec at implementation depth; none is silently dropped as "post-MVP".
- Enhancement behavior matches the prototype where the prototype fixes it (copy, defaults such as mental-health-off, durations 24 hours / 30 days / 90 days, demo data).

---

## 5. MVP scope

### 5.1 MVP capabilities (PRD §30)

| Area | MVP capability | Prototype evidence |
|------|----------------|--------------------|
| Identity | Patient registration; doctor registration/authentication; hospital/organization registration | Patient welcome + OTP; Doctor login; Admin dashboard |
| Profile | Patient profile | Patient profile screen |
| QR | Patient QR generation; doctor QR scanning; authorized patient lookup | Patient QR present; Doctor QR scan |
| Clinical view | Patient overview; medical timeline; diagnosis records; prescription records | Patient home, Patient timeline; Doctor patient 360 |
| Documents | Report/document upload; report/document viewing; discharge summary | Doctor add to record; Patient records, Report detail, Discharge detail |
| Control | Basic consent/access control; audit logging | Consent request, Access and consent; audit toasts throughout |
| Integration | Hospital integration onboarding; secure integration API; patient identity mapping; finalized report synchronization; patient notification when a synchronized report becomes available | Admin dashboard, Admin integration queue; Notifications screen |

The prototype covers this full MVP journey end-to-end across its 21 screens: patient (welcome → OTP → home → QR → timeline → records → report detail → profile → access and consent → consent request → notifications → emergency card → discharge detail), doctor (login → home/search → scan or break-glass → 360 → add to record), admin (dashboard → integration queue).

### 5.2 Explicit deferrals (PRD §30) and their reconciliation with the prototype

The PRD deferral list predates the approved enhancements. Reconciled status:

| PRD deferral | Status after gap analysis / prototype |
|--------------|---------------------------------------|
| Advanced AI/diagnostic capabilities | **Partially superseded.** AI *summarization* (patient summary, clinical brief, ask-the-record, per-report explanation) is now top-priority and in the MVP. AI *diagnosis* remains out of scope — the AI never diagnoses, and every AI surface carries the disclaimers in [05-ai-features.md](05-ai-features.md). |
| Automated clinical decision support | **Narrowly superseded.** The allergy conflict warning (#14) is a deterministic safety check on known allergy classes (penicillin-class demo), not a general CDS engine. General CDS stays deferred. |
| Insurance integration | Deferred. Not in prototype. |
| Pharmacy integration | Deferred. Not in prototype. |
| Complex healthcare billing | Deferred. Not in prototype. |
| Advanced analytics | Deferred. The HbA1c/vitals trend chart (#11) is a per-patient display of existing results, not an analytics capability. |
| Full interoperability implementation | Deferred, with alignment. The MVP is FHIR-friendly (Apollo LIS integrates via FHIR; ABDM/ABHA alignment #18) but full FHIR conformance is future work — see [12-api-and-interoperability.md](12-api-and-interoperability.md). |
| Multi-country compliance abstractions | Deferred. India-first deployment (DPDP, ABDM); architecture keeps HIPAA/GDPR reachable — see [13-non-functional.md](13-non-functional.md). |

### Acceptance criteria

- Every MVP capability row has a specified implementation in the sibling specs.
- No sibling spec designs insurance, pharmacy, billing, or general-purpose analytics/CDS features.
- Every AI surface in the product is a summarization/Q&A surface with disclaimers; none produces a diagnosis or treatment recommendation.

---

## 6. Core design principle

From PRD §33, verbatim:

> **The QR code identifies the patient; authentication and authorization determine what the doctor can see.**

The QR code is an access mechanism, **not an authentication mechanism**. It never contains medical information — only a secure identifier/token resolved by Atlas, with expiration, rotation, revocation, session binding, rate limiting, and anti-replay protection (PRD §10). The full token design is in [08-qr-subsystem.md](08-qr-subsystem.md); the authorization decision it feeds is in [07-consent-and-access-control.md](07-consent-and-access-control.md).

Consequences that bind every spec:

- Scanning a QR alone grants nothing. The doctor must be authenticated, and the consent/authorization evaluation must pass, before any data is shown.
- Losing a phone (or a screenshot of a QR) must not leak medical data.
- Every QR-mediated access is written to the audit trail (PRD §§10, 13).

---

## 7. Future vision

From PRD §34: Atlas can evolve from a medical-record repository into a **portable health identity and interoperability layer**. Potential future capabilities (not specified further in this set, but the architecture must not preclude them):

- Patient-controlled health identity; cross-hospital medical history.
- FHIR-based interoperability; health data import/export.
- Lab, pharmacy, and insurance integrations.
- AI-assisted medical-history summarization (now pulled forward — see §5.2).
- Drug interaction detection (the allergy warning #14 is a first step).
- Longitudinal health analytics; patient health timeline.
- Emergency medical profile (the emergency card #10 is a first step).
- Consent marketplace / delegated access; healthcare provider network.

PRD §29 sets the interoperability posture: FHIR compatibility is not required in the first MVP, but the data model must not make future adoption of FHIR, standardized diagnosis/laboratory codes, or medication identifiers unnecessarily difficult. [09-data-model.md](09-data-model.md) and [12-api-and-interoperability.md](12-api-and-interoperability.md) carry this constraint.

---

## 8. Success criteria

From PRD §35, normalized into two groups. (The PRD lists the integration criteria, numbered 11–16, above the MVP criteria 1–10; the intended reading is one list of 16.)

**Atlas MVP is successful when:**

| # | Criterion |
|---|-----------|
| 1 | A patient can create and maintain a digital health profile. |
| 2 | A patient can present a QR code. |
| 3 | An authenticated doctor can scan the QR code. |
| 4 | Atlas securely identifies the patient. |
| 5 | The doctor can access only information they are authorized to see. |
| 6 | The doctor can understand the patient's relevant medical history from a consolidated view. |
| 7 | New medical information can be added during a consultation. |
| 8 | Reports and discharge summaries can be securely stored and retrieved. |
| 9 | Every sensitive access is auditable. |
| 10 | The architecture can evolve toward healthcare interoperability without a fundamental redesign. |

**Integration success criteria:**

| # | Criterion |
|---|-----------|
| 11 | A hospital can integrate Atlas with its existing EHR/LIS without replacing that system. |
| 12 | A finalized laboratory report can flow from the hospital system into Atlas. |
| 13 | Atlas can correctly associate synchronized clinical data with the intended patient. |
| 14 | Patients can be notified when eligible hospital-generated reports become available. |
| 15 | Synchronized clinical data retains source provenance and auditability. |
| 16 | Integration failures are detectable, retryable, and do not silently lose clinical information. |

Proposed: in addition, the 18 approved enhancements (§4) and the AI summary surfaces are treated as MVP acceptance scope, since they were approved into the build and are in the prototype.

---

## 9. Platform targets

The mobile app must ship on **both iOS and Android**. The prototype renders every screen in both device frames via a toggle:

| Platform | Frame reference | Design language |
|----------|-----------------|-----------------|
| iOS | `design/ios-frame.jsx` | iOS 26 "liquid glass" chrome |
| Android | `design/android-frame.jsx` | Material 3 chrome |

Screen content is a single shared design (390 × 844 logical canvas in the prototype) rendered inside either frame; only device chrome differs. Platform-specific behavior that the specs call out explicitly: biometric login is Face ID on iOS and the platform biometric on Android (see [06-identity-auth.md](06-identity-auth.md)); the offline emergency card targets the lock screen on both platforms (see [02-patient-app.md](02-patient-app.md)).

Visual direction is **"1a Meridian"** — editorial calm: Instrument Serif for display headings, Instrument Sans for body, warm paper background (`#faf7f1` screens on an `#efece5` canvas), ink `#22303c`, ocean-blue accent `oklch(0.52 0.13 235)`, compass-ring logo. The full token set and component rules are in [01-design-system.md](01-design-system.md).

Admin surfaces are shown in the same mobile prototype; the architecture document additionally specifies an Admin web console (staff and role management, integration configuration and credentials, error queue, publication rules, audit export) — detailed in [04-admin-app.md](04-admin-app.md) and [10-architecture.md](10-architecture.md).

### Acceptance criteria

- Feature parity: every specified screen and flow exists on both iOS and Android.
- Both apps implement the Meridian design system tokens from [01-design-system.md](01-design-system.md); platform chrome (navigation transitions, system controls) follows each platform's conventions.

---

## 10. Glossary

Canonical vocabulary for the spec set. Sibling specs use these terms without redefining them.

| Term | Definition |
|------|------------|
| **Atlas ID** | The globally unique, platform-level patient identifier, e.g. `ATL-82X92K`. Independent of any hospital's own patient ID (PRD §22). |
| **Atlas QR** | The patient's presentable QR code. Encodes only a secure, resolvable token — never medical data. Supports expiration, rotation, revocation, session binding, rate limiting, anti-replay (PRD §10; [08-qr-subsystem.md](08-qr-subsystem.md)). |
| **Tenant** | A healthcare organization (hospital, clinic chain) as an isolation and ownership boundary in the multi-tenant SaaS. Clinical records are tenant-owned; tenant context is resolved server-side on every request and never trusted from the client (Canvas). |
| **Tenancy tier** | How a tenant's data is isolated: pooled (shared tables + `tenant_id` + row-level security), siloed (schema-per-tenant), or dedicated (DB/cluster-per-tenant) (Canvas; [10-architecture.md](10-architecture.md)). |
| **360-degree view (patient 360)** | The doctor's consolidated view of a patient's history composed at read time across tenants, filtered by the active consent (PRD §14; [03-doctor-app.md](03-doctor-app.md)). |
| **Consent** | A patient-granted permission for a specific doctor/organization to read defined categories of the patient's record for a bounded duration (24 hours / 30 days / 90 days in the prototype). Revocable at any time ([07-consent-and-access-control.md](07-consent-and-access-control.md)). |
| **Consent category** | A class of information toggled independently in a grant: general history, reports & labs, medications, diagnoses, mental/behavioral health (off by default), and sensitive clinical data (PRD §12; enhancement #2/#8). The prototype's grant screen exposes four as toggles: General history, Reports & labs, Medications, Mental health. |
| **Break-glass** | Emergency access path letting a doctor open a record without prior consent under a mandatory declared emergency reason; access is limited to the emergency profile plus critical history, compliance and the patient are alerted immediately, and the session is heavily audited ([07-consent-and-access-control.md](07-consent-and-access-control.md)). |
| **Provenance** | Retained origin metadata on every synchronized record: source system, source organization, source record ID, source timestamp, imported-at, record status. Never overwritten by transformation (PRD §27). |
| **Report lifecycle** | The status progression of a report: `ORDERED → IN_PROGRESS → COMPLETED → VERIFIED → FINAL → PUBLISHED_TO_PATIENT` (PRD §24), surfaced in the UI as Final / Preliminary / Amended (enhancement #3). |
| **Publication policy** | Per-tenant configuration of when a report type becomes patient-visible: auto after finalization, after verification, held for manual release, or restricted to professionals (PRD §24). |
| **Identity resolution** | Mapping an external hospital patient ID (e.g. `HOSP-928372`) to an Atlas ID, with confidence checks; ambiguous matches go to human resolution, never auto-attach (PRD §22). |
| **Integration error queue** | The admin-visible queue of failed synchronization events with reasons and retry, backed by the processing states `RECEIVED / VALIDATING / PROCESSED / REJECTED / RETRYING / FAILED / DEAD_LETTER` (PRD §26; enhancement #5). |
| **ABHA** | Ayushman Bharat Health Account — India's national health ID (e.g. `priya@abdm`), federated with the Atlas patient identity in India deployments. |
| **ABDM** | Ayushman Bharat Digital Mission — the Indian national digital health ecosystem Atlas aligns with (enhancement #18). |
| **Family profile** | A dependent's profile (e.g. Aarav, Kamala Sharma) managed under one patient login, each with its own Atlas ID and record (enhancement #7). |
| **Emergency card** | The offline, lock-screen-accessible card carrying life-critical facts (blood group, allergies, conditions, current medications, emergency contact) without authentication (enhancement #10). |
| **AI health summary** | Plain-language, patient-facing summary of the record shown on the patient home, with disclaimers ([05-ai-features.md](05-ai-features.md)). |
| **AI clinical brief** | Doctor-facing summary generated at QR-scan time to orient the consultation; carries "verify before clinical use" disclaimers ([05-ai-features.md](05-ai-features.md)). |
| **Ask-the-record** | Follow-up Q&A over the patient's own record via suggested chips or free questions, grounded in record data ([05-ai-features.md](05-ai-features.md)). |
| **AI report explanation** | Per-report plain-language explanation of values and reference ranges for patients, generated on demand from the report's own data ([05-ai-features.md](05-ai-features.md) §5). |
| **Timeline event** | One entry in the chronological medical timeline: consultation, diagnosis, prescription, lab, imaging, procedure, surgery, admission, discharge, referral, vaccination, follow-up, or another clinical event (PRD §5). |
| **Audit trail** | Tamper-resistant log of every sensitive operation, with actor, role, patient, organization, timestamp, action, resource, access method, result, and session info (PRD §13). |
| **System of record** | The hospital's own EHR/HIS/LIS remains authoritative for clinical operations; Atlas aggregates and controls access, and hospitals never write directly to the Atlas core database (PRD §19). |
