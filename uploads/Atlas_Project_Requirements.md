# Atlas — Project Requirements

## 1. Overview

**Atlas** is a secure, patient-centric digital health record platform that allows a patient's medical information to be stored in one place and made available to authenticated healthcare professionals through a controlled QR-based access mechanism.

The core concept is:

> **One patient → One identity → One QR → Complete medical history → Controlled doctor access**

Atlas should provide doctors with a consolidated view of relevant patient information while ensuring that access is authenticated, authorized, auditable, and privacy-conscious.

---

## 2. Goals

- Maintain a centralized digital record for each patient.
- Make a patient's medical history available across healthcare encounters.
- Allow authenticated doctors to quickly access patient information using a QR code.
- Give doctors a consolidated view of the patient's medical journey.
- Support structured medical information as well as uploaded reports/documents.
- Maintain a complete audit trail of access and changes.
- Ensure patients retain control over who can access their information.
- Design the platform so it can evolve into a broader healthcare interoperability platform.

---

## 3. Core Users

### 3.1 Patient

A patient owns or is associated with a digital Atlas profile containing their medical history.

Patients should be able to:

- Register/create their profile.
- View their medical information.
- View medical reports and documents.
- View diagnoses and treatment history.
- View prescriptions and discharge summaries.
- Generate/display their Atlas QR code.
- See which doctors or healthcare organizations have accessed their records.
- Grant or revoke access where applicable.
- Manage basic profile information.

### 3.2 Doctor / Healthcare Professional

Doctors use Atlas to retrieve patient information during a consultation.

Doctors should be able to:

- Authenticate securely.
- Scan a patient's Atlas QR code.
- Request/access patient information according to authorization rules.
- View a consolidated patient profile.
- Review previous diagnoses, consultations, reports, prescriptions, procedures, and discharge summaries.
- Add new clinical information.
- Upload medical documents/reports.
- Record consultations and diagnoses.
- View relevant historical information before making clinical decisions.

### 3.3 Healthcare Organization / Admin

An organization-level role may be introduced to manage:

- Doctors and staff.
- Organization profiles.
- Access policies.
- Patient records created by the organization.
- Audit logs.
- Organization-level configuration.

---

# 4. Patient Profile

Each patient should have a unique Atlas identity.

### Basic information

- Atlas Patient ID
- Full name
- Date of birth
- Gender
- Contact information
- Address
- Emergency contact
- Blood group
- Allergies
- Existing conditions
- Other relevant demographic information

Sensitive information should only be exposed according to the user's authorization and the applicable access policy.

---

# 5. Medical History

Atlas should maintain a chronological medical timeline.

Possible timeline events include:

- Doctor consultation
- Diagnosis
- Prescription
- Laboratory test
- Imaging/report
- Procedure
- Surgery
- Hospital admission
- Hospital discharge
- Referral
- Vaccination
- Follow-up
- Other clinical events

Each event should contain appropriate metadata such as:

- Date/time
- Healthcare provider
- Organization
- Event type
- Diagnosis/reason
- Clinical notes
- Attachments
- Related prescriptions
- Related reports

---

# 6. Medical Documents & Reports

Atlas should support storing medical documents and reports.

Examples:

- Blood test reports
- Urine test reports
- X-ray reports
- CT reports
- MRI reports
- Ultrasound reports
- ECG reports
- Pathology reports
- Prescription documents
- Discharge summaries
- Referral letters
- Operation/procedure reports
- Other medical documents

Requirements:

- Documents should be associated with the relevant patient and medical event.
- Documents should have metadata.
- Documents should be securely stored.
- Access should be authorization-controlled.
- Document access should be auditable.
- The system should support common document/image formats.

---

# 7. Diagnosis

Atlas should maintain structured diagnosis information where possible.

A diagnosis record may include:

- Diagnosis
- Diagnosis code
- Date diagnosed
- Diagnosing doctor
- Organization
- Status
- Notes
- Related reports
- Related treatment

The architecture should allow standardized medical terminology/coding to be introduced later.

---

# 8. Prescriptions & Medication History

The system should maintain medication history.

Each prescription may contain:

- Medication name
- Dosage
- Frequency
- Route
- Duration
- Instructions
- Prescribing doctor
- Date prescribed
- Start/end date
- Status

The system should distinguish between active, completed, discontinued, and historical medications.

---

# 9. Hospitalization & Discharge

Atlas should support hospitalization records.

A hospitalization record may include:

- Hospital/organization
- Admission date
- Discharge date
- Reason for admission
- Diagnoses
- Procedures
- Treatments
- Doctors involved
- Medications
- Investigations
- Discharge condition
- Follow-up instructions
- Discharge summary
- Attachments

---

# 10. QR-Based Patient Access

QR is one of Atlas's primary interaction mechanisms.

Each patient should have an Atlas QR identity.

### Basic flow

1. Patient opens Atlas.
2. Patient displays their QR code.
3. Doctor authenticates into Atlas.
4. Doctor scans the patient's QR code.
5. Atlas identifies the patient.
6. Atlas validates the doctor's authorization.
7. Patient access/consent rules are evaluated.
8. Authorized information is displayed.
9. The access event is recorded in the audit log.

### Important security principle

The QR code should **not contain the patient's medical information**.

It should contain only a secure identifier/token that can be resolved by Atlas.

QR codes should also support mechanisms such as:

- Expiration
- Rotation
- Revocation
- Session binding
- Rate limiting
- Anti-replay protection

---

# 11. Authentication

### Doctor authentication

Doctors should authenticate using secure mechanisms such as:

- Email/username + password
- MFA
- Organization identity provider
- Future support for professional identity verification

### Patient authentication

Patients should have a secure authentication mechanism appropriate to the platform.

Potential options:

- Mobile number + OTP
- Email authentication
- Password
- Passkeys
- MFA

Authentication and authorization must be treated separately.

---

# 12. Authorization & Consent

Access to medical information must be controlled.

Atlas should support:

- Role-based access control.
- Patient-level authorization.
- Organization-level permissions.
- Doctor-level permissions.
- Consent-based access.
- Temporary access.
- Access expiration.
- Revocation.
- Emergency access policies where required.

Different types of information may eventually have different access policies.

For example:

- General medical history
- Medications
- Diagnoses
- Laboratory reports
- Mental/behavioral health information
- Sensitive clinical information

---

# 13. Audit Trail

Every sensitive operation should be auditable.

The system should record events such as:

- Patient record viewed
- Document viewed
- Patient accessed via QR
- Access granted
- Access revoked
- Record created
- Record modified
- Document uploaded
- Document downloaded
- Diagnosis added
- Prescription added

Audit records should contain appropriate metadata such as:

- User
- Role
- Patient
- Organization
- Timestamp
- Action
- Resource
- Access method
- Result
- Relevant request/session information

Audit logs should be tamper-resistant and protected from unauthorized modification.

---

# 14. Patient Timeline / 360° View

The doctor should have a single consolidated patient view.

Example structure:

```text
Patient
│
├── Overview
│   ├── Basic Information
│   ├── Allergies
│   ├── Existing Conditions
│   └── Current Medications
│
├── Timeline
│   ├── Consultations
│   ├── Diagnoses
│   ├── Procedures
│   ├── Hospitalizations
│   └── Follow-ups
│
├── Reports
│   ├── Laboratory
│   ├── Imaging
│   └── Other Reports
│
├── Prescriptions
│
├── Discharge Summaries
│
└── Documents
```

The UI should prioritize information that is clinically relevant to the current consultation.

---

# 15. Search

Authorized users should be able to search patient records according to their permissions.

Potential search capabilities:

- Patient ID
- Name
- Phone number
- Date of birth
- QR scan
- Organization-specific patient identifier

Search results must not expose medical information to unauthorized users.

---

# 16. Notifications

Potential notifications include:

- New medical record added
- New report uploaded
- Access granted
- Access revoked
- Doctor accessed records
- New prescription
- Follow-up reminder
- Consent request

Notifications should be configurable and should not expose sensitive medical information unnecessarily.

---

# 17. Security Requirements

Security is a core requirement rather than a later enhancement.

Atlas should include:

- Encryption in transit.
- Encryption at rest.
- Strong authentication.
- Role-based authorization.
- Fine-grained access control.
- Secure session management.
- Secure QR token design.
- Rate limiting.
- Protection against replay attacks.
- Input validation.
- Secure file handling.
- Malware scanning for uploaded files.
- Comprehensive audit logging.
- Secrets management.
- Backup and disaster recovery.
- Data retention policies.
- Secure deletion where legally required.

---

# 18. Privacy & Compliance

Atlas is expected to handle highly sensitive personal and medical information.

The architecture should therefore be designed with privacy and regulatory compliance in mind from the beginning.

The implementation should identify and accommodate the applicable regulations based on the deployment geography, such as:

- India's Digital Personal Data Protection framework.
- Applicable health-data regulations.
- HIPAA where applicable to US deployments.
- GDPR where applicable to EU/EEA users.

Compliance requirements should be validated separately with appropriate legal/compliance expertise before production deployment.

---


# 19. Hospital & EHR Integration

Atlas should allow hospitals and healthcare organizations to integrate their existing EHR/HIS/LIS systems with Atlas without requiring them to replace their current systems.

The hospital should remain the system of record for clinical operations while Atlas acts as a secure patient-centric aggregation and access platform.

### Primary integration flow

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

### Example: Laboratory Report

1. Doctor orders a laboratory test in the hospital's existing EHR.
2. The order is processed by the hospital/LIS.
3. The laboratory completes the test.
4. The LIS/EHR marks the report as verified/finalized.
5. The hospital system sends the finalized result/report to Atlas.
6. Atlas validates the source and resolves the patient identity.
7. Atlas stores the report and associates it with the patient timeline.
8. Atlas publishes the report according to the patient's access and visibility policy.
9. The patient receives a notification that a new report is available.
10. The patient can view the report through Atlas.
11. Authorized doctors can view the report as part of the patient's medical history.

The same integration model should support finalized diagnoses, prescriptions, imaging reports, discharge summaries, procedures, and other supported clinical information.

### Integration boundary

Hospitals should **not write directly to the Atlas core database**.

All external data should pass through the Atlas integration layer:

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

This creates a stable boundary between Atlas and hospital systems and allows multiple hospitals to integrate independently.

---

# 20. Integration Methods

Atlas should support multiple integration mechanisms because hospitals may have very different technology stacks.

Potential mechanisms include:

- REST APIs
- HL7 FHIR APIs
- HL7 v2 messaging
- Webhooks / event-driven integration
- Secure file-based integration for legacy systems
- Scheduled synchronization where real-time events are unavailable

For the initial architecture, prioritize:

1. REST APIs
2. FHIR-compatible interfaces
3. Webhooks / event notifications

The integration layer should remain extensible so additional protocols can be introduced without changing Atlas's core medical-record model.

---

# 21. Hospital Integration Configuration

A hospital administrator should be able to configure an Atlas integration.

Configuration may include:

- Hospital/organization identity
- Integration credentials
- API endpoints
- Supported integration protocol
- Patient identifier mapping
- Supported clinical resources
- Report publication rules
- Notification preferences
- Synchronization mode
- Webhook configuration
- Integration status
- Error/retry configuration

Integration credentials and secrets must never be exposed to ordinary users.

---

# 22. Patient Identity Resolution

A hospital may have its own patient identifier while Atlas has a separate Atlas Patient ID.

Example:

```text
Hospital Patient ID: HOSP-928372
                    │
                    ▼
             Identity Resolution
                    │
                    ▼
Atlas Patient ID: ATL-82X92K
```

Atlas must maintain a secure identity mapping between external identifiers and Atlas Patient IDs.

The system should prevent duplicate Atlas patient records where an existing patient can be confidently identified.

Identity matching should support appropriate matching attributes and confidence checks. Ambiguous matches should be routed for explicit resolution rather than automatically attaching clinical information to the wrong patient.

Every identity-linking operation should be auditable.

---

# 23. Clinical Data Synchronization

Atlas should support synchronization of selected clinical information from integrated hospital systems.

Potential resources include:

- Patient demographics
- Encounters
- Diagnoses
- Laboratory orders
- Laboratory results
- Imaging orders
- Imaging reports
- Prescriptions
- Medications
- Procedures
- Hospital admissions
- Discharges
- Discharge summaries
- Clinical documents
- Referrals
- Other supported clinical events

The synchronization model should preserve:

- Source system
- Source record identifier
- Original clinical timestamp
- Author/provider
- Organization
- Record status
- Provenance
- Version where applicable

Atlas should distinguish between information originating in Atlas and information synchronized from an external system.

---

# 24. Report Lifecycle & Publication

Atlas should model report status so that incomplete or preliminary information is not unintentionally presented as a final clinical report.

A typical lifecycle is:

```text
ORDERED
   ↓
IN_PROGRESS
   ↓
COMPLETED
   ↓
VERIFIED
   ↓
FINAL
   ↓
PUBLISHED_TO_PATIENT
```

Only reports meeting the configured publication criteria should become visible to patients.

Hospitals should be able to configure whether specific report types are:

- Automatically published after finalization
- Published after verification
- Held for manual release
- Restricted to healthcare professionals

If a report is corrected or amended after publication, Atlas should retain appropriate version/provenance information and clearly indicate that an updated version is available.

---

# 25. Event-Driven Integration

Atlas should support event-driven synchronization where the source system can notify Atlas when clinical information changes.

Example:

```text
Lab finalizes report
        ↓
Hospital EHR/LIS emits event
        ↓
Atlas Integration Gateway
        ↓
Validate source
        ↓
Resolve patient
        ↓
Validate report
        ↓
Store report
        ↓
Update patient timeline
        ↓
Apply publication policy
        ↓
Notify patient
```

The integration layer should support:

- Idempotent event processing
- Retry handling
- Dead-letter/error handling
- Duplicate event detection
- Event ordering where required
- Source-system acknowledgement
- Processing status
- Integration monitoring

---

# 26. Integration Error Handling

Failed integrations must not silently lose clinical information.

Atlas should maintain an integration processing state such as:

- RECEIVED
- VALIDATING
- PROCESSED
- REJECTED
- RETRYING
- FAILED
- DEAD_LETTER

Hospital administrators should be able to identify failed synchronization events and understand the reason for failure without exposing sensitive information unnecessarily.

---

# 27. Provenance

Every synchronized clinical record should retain its source provenance.

For example:

```text
Source System: Hospital EHR
Source Organization: Example Hospital
Source Record ID: LAB-928372
Source Timestamp: 2026-08-21T10:30:00Z
Imported At: 2026-08-21T10:31:02Z
Record Status: FINAL
```

Atlas should not overwrite source provenance when records are transformed or normalized.

This is important for auditability, clinical trust, reconciliation, and future interoperability.


# 28. API-First Architecture

Atlas should expose APIs for major platform capabilities.

Potential API domains:

```text
/auth
/patients
/doctors
/organizations
/encounters
/diagnoses
/prescriptions
/medications
/reports
/documents
/hospitalizations
/discharge-summaries
/consents
/access
/audit
/qr
/notifications
```

The API layer should support future integration with:

- Hospitals
- Diagnostic laboratories
- Pharmacies
- Insurance providers
- Healthcare applications
- National/regional health platforms
- Electronic Health Record systems

---

# 29. Interoperability

The data model should avoid becoming tightly coupled to Atlas-specific representations.

The architecture should allow future adoption of healthcare interoperability standards such as:

- HL7 FHIR
- Standardized diagnosis codes
- Standardized laboratory codes
- Standardized medication identifiers

FHIR compatibility does not have to be part of the first MVP, but the initial data model should avoid making future interoperability unnecessarily difficult.

---

# 30. MVP Scope

The first version should focus on the simplest useful end-to-end journey.

### MVP capabilities

- Patient registration
- Doctor registration/authentication
- Hospital/organization registration
- Patient profile
- Patient QR generation
- Doctor QR scanning
- Authorized patient lookup
- Patient overview
- Medical timeline
- Diagnosis records
- Prescription records
- Report/document upload
- Report/document viewing
- Discharge summary
- Basic consent/access control
- Audit logging
- Hospital integration onboarding
- Secure integration API
- Patient identity mapping
- Finalized report synchronization
- Patient notification when a synchronized report becomes available

### Explicitly defer from MVP

- Advanced AI/diagnostic capabilities
- Insurance integration
- Pharmacy integration
- Complex healthcare billing
- Advanced analytics
- Full interoperability implementation
- Multi-country compliance abstractions
- Automated clinical decision support

---

# 31. Non-Functional Requirements

### Performance

- QR-to-patient lookup should feel near-instantaneous under normal conditions.
- Patient overview should load quickly even for patients with extensive history.
- Document access should support large files without blocking the main application.

### Availability

Atlas should be designed for high availability because doctors may depend on the system during active patient care.

### Scalability

The platform should support scaling independently for:

- API services
- Database
- Document storage
- Authentication
- Search
- Audit logging

### Reliability

- Durable medical records.
- Automated backups.
- Disaster recovery.
- Data integrity checks.
- Safe migration mechanisms.

---

# 32. Suggested High-Level Architecture

```text
                    ┌─────────────────┐
                    │     Patient     │
                    │   Atlas App     │
                    └────────┬────────┘
                             │
                         QR Code
                             │
                             ▼
┌───────────────┐      ┌───────────────┐
│     Doctor    │─────▶│  Atlas API    │
│ Web / Mobile  │      └───────┬───────┘
└───────────────┘              │
                               ▼
                    ┌────────────────────┐
                    │ Authentication &    │
                    │ Authorization       │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
       ┌────────────┐  ┌─────────────┐  ┌──────────────┐
       │ Patient &  │  │ Medical     │  │ Audit &      │
       │ Identity   │  │ Records     │  │ Access Logs  │
       └────────────┘  └─────────────┘  └──────────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │  Document   │
                       │   Storage   │
                       └─────────────┘
```

---

# 33. Core Design Principle

Atlas should be built around one fundamental principle:

> **The QR code identifies the patient; authentication and authorization determine what the doctor can see.**

The QR code is therefore an access mechanism, **not an authentication mechanism**.

This distinction is critical for security and privacy.

---

# 34. Future Vision

Atlas can evolve from a digital medical-record repository into a **portable health identity and interoperability layer**.

Potential future capabilities:

- Patient-controlled health identity
- Cross-hospital medical history
- FHIR-based interoperability
- Health data import/export
- Lab integrations
- Pharmacy integrations
- Insurance integrations
- AI-assisted medical-history summarization
- Drug interaction detection
- Longitudinal health analytics
- Patient health timeline
- Emergency medical profile
- Consent marketplace / delegated access
- Healthcare provider network

---

## 35. Success Criteria

11. A hospital can integrate Atlas with its existing EHR/LIS without replacing that system.
12. A finalized laboratory report can flow from the hospital system into Atlas.
13. Atlas can correctly associate synchronized clinical data with the intended patient.
14. Patients can be notified when eligible hospital-generated reports become available.
15. Synchronized clinical data retains source provenance and auditability.
16. Integration failures are detectable, retryable, and do not silently lose clinical information.


Atlas MVP will be considered successful when:

1. A patient can create and maintain a digital health profile.
2. A patient can present a QR code.
3. An authenticated doctor can scan the QR code.
4. Atlas securely identifies the patient.
5. The doctor can access only information they are authorized to see.
6. The doctor can understand the patient's relevant medical history from a consolidated view.
7. New medical information can be added during a consultation.
8. Reports and discharge summaries can be securely stored and retrieved.
9. Every sensitive access is auditable.
10. The architecture can evolve toward healthcare interoperability without a fundamental redesign.
