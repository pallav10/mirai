# Atlas — Specifications

Atlas is a secure, patient-centric digital health record platform: one patient → one identity → one QR → complete medical history → controlled doctor access. It is a multi-tenant SaaS (tenant = healthcare organization) with globally-scoped patient identity, tenant-owned clinical records, a consented read-time 360° composition across tenants, three roles (Patient, Doctor, Admin), and top-priority AI summarization surfaces. This spec set was produced from the Claude Design handoff bundle in `design/` — the PRD, the clickable hi-fi prototype, the architecture document, the system-design canvas, the component-level LLD, and the chosen visual direction ("1a Meridian") — and normalizes all of it into fourteen implementation-ready documents. Prototype-grounded behavior is stated as fixed; extrapolation beyond the sources is labeled "Proposed".

## Spec index

| File | Title | Description |
|---|---|---|
| [00-product-overview.md](00-product-overview.md) | Product overview | What Atlas is, who it serves, the 18 approved post-PRD enhancements, MVP scope, success criteria, and the glossary every other spec uses. |
| [01-design-system.md](01-design-system.md) | Design system | The Meridian visual language: tokens (paper/ink/ocean, oklch palette), Instrument Serif/Sans typography, component rules, motion, and accessibility standards shared by all three apps. |
| [02-patient-app.md](02-patient-app.md) | Patient app | Every patient-facing screen, state, and interaction: onboarding/OTP, home with AI summary, QR present, timeline, records and report detail, family profiles, consent flows, notifications, emergency card. |
| [03-doctor-app.md](03-doctor-app.md) | Doctor app | The clinician app: org SSO + MFA login, patient search, QR scan consults, break-glass, the patient 360° with AI clinical brief and ask-the-record, clinical writes with allergy checks, and the consult session model. |
| [04-admin-app.md](04-admin-app.md) | Admin app | The organization-admin console: org dashboard (KPIs, integration health), the integration error queue, and the Proposed admin capabilities — staff, policies, audit export, compliance review, static QR issuance. |
| [05-ai-features.md](05-ai-features.md) | AI features | The four MVP AI capabilities (patient summary, clinical brief, ask-the-record, report explanation), the authorize→retrieve→verify RAG pipeline, provider gateway, caching, and the safety/evaluation regime. |
| [06-identity-auth.md](06-identity-auth.md) | Identity & authentication | Who exists in Atlas (patient, practitioner, organization, system client), how each identity is proven — OTP/passkey/biometric, org IdP + MFA, mTLS — and how sessions are established and audited. |
| [07-consent-and-access-control.md](07-consent-and-access-control.md) | Consent & access control | The policy engine (RBAC × ABAC × consent), the ConsentGrant lifecycle and every grant path, revocation, break-glass emergency access, the lock-screen emergency card, and the audit-trail event catalog. |
| [08-qr-subsystem.md](08-qr-subsystem.md) | QR subsystem | The opaque rotating QR token design (300 s TTL, anti-replay, session binding), present and scan experiences, the end-to-end scan flow, uniform failure handling, fallbacks, and the threat model. |
| [09-data-model.md](09-data-model.md) | Domain data model | Every persistent entity — Patient, TenantOrganization, ConsentGrant, TimelineEvent and subtypes, Report lifecycle, Provenance, AuditEvent, IntegrationEvent — with ownership, lifecycles, and the canonical persona dataset. |
| [10-architecture.md](10-architecture.md) | System architecture | End-to-end topology: edge/gateway tenant resolution, core services, eventing, data layer, the multi-tenancy model (pooled/siloed/dedicated tiers and invariants), key flows, deployment, and the MVP boundary. |
| [11-integrations.md](11-integrations.md) | Hospital & EHR integration | The one-way integration boundary and pipeline: methods (FHIR/webhooks/file), per-tenant connector configuration, identity resolution, report lifecycle and publication, error handling states, and provenance. |
| [12-api-and-interoperability.md](12-api-and-interoperability.md) | API surface & interoperability | The versioned REST route catalog for all clients, the error model, pagination and idempotency, FHIR R4 alignment, ABDM/ABHA integration points, export formats, and the outbound webhook contract. |
| [13-non-functional.md](13-non-functional.md) | Non-functional requirements | Platform-wide security, privacy/compliance (DPDP/HIPAA/GDPR), performance SLOs, availability and DR, observability, accessibility, localization (English/Hindi/Marathi), and operational requirements. |

## How to read

1. Start with **[00-product-overview.md](00-product-overview.md)** — vision, roles, enhancement list, MVP scope, and the shared vocabulary.
2. Then **[01-design-system.md](01-design-system.md)** — the visual system every screen spec assumes.
3. Then the app specs, **02 → 03 → 04** (patient, doctor, admin) — the product surface, screen by screen.
4. Then the platform specs, **05 → 13** — AI, identity, consent, QR, data model, architecture, integrations, API, and non-functional requirements. Each names its owning topics; where two specs touch the same topic, the owner named in the text is authoritative.

## Source material

The specs derive from the design bundle in `design/`:

| File | Contents |
|---|---|
| `design/uploads/Atlas_Project_Requirements.md` | The PRD — 35 numbered sections; authoritative for requirements. |
| `design/Atlas App.dc.html` | The clickable hi-fi prototype (21 screens, three roles, logic script); authoritative for screens, layout, copy, colors, and interactions. |
| `design/Atlas Architecture.dc.html` | The written end-to-end architecture document (incl. §10 AI architecture); authoritative for backend architecture. |
| `design/Canvas.dc.html` | The high-level multi-tenant system design diagram: layers, tenancy tiers, invariants, key flows. |
| `design/Atlas LLD.dc.html` | Component-level design: UML component diagram, two sequence diagrams, ingest activity flowchart, and the 11-entity domain class model. |
| `design/Atlas Directions.dc.html` | The three visual directions explored; direction "1a Meridian" was chosen. |
| `design/ios-frame.jsx` / `design/android-frame.jsx` | Device-frame mock components (iOS 26 liquid glass; Material 3) used by the prototype. |
