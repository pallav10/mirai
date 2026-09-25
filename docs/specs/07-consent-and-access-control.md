# 07 — Consent & Access Control

Defines how Atlas decides who may see what: the policy engine, consent as a first-class object, every grant path, revocation, break-glass emergency access, the emergency lock-screen card, and the audit trail that makes all of it accountable.

Sources: `design/uploads/Atlas_Project_Requirements.md` (PRD §12 Authorization & Consent, §13 Audit Trail, §17 Security Requirements, §18 Privacy & Compliance, §33 Core Design Principle), `design/Atlas Architecture.dc.html` (§2 principles, §7 Authorization and consent, §9 Consent & access + Audit services, §10.2/10.4 AI authorization, §12 eventing, §16 flows C and D), `design/Canvas.dc.html` (identity/authz/consent layer, multi-tenancy invariants), `design/Atlas LLD.dc.html` (Policy Engine and Consent Service components, QR-consult sequence, `ConsentGrant` and `AuditEvent` classes), `design/Atlas App.dc.html` (Access & consent, Consent request, Emergency card, break-glass, doctor 360 screens and interaction logic).

---

## 1. Principles and scope

Atlas is built around one rule (PRD §33): **the QR code identifies the patient; authentication and authorization determine what the doctor can see.** The QR is an access mechanism, never an authentication mechanism, and never carries medical data.

From the architecture principles (Architecture §2) and tenancy invariants (Architecture §3.3):

- **Consent is a first-class object** — scoped by grantee, category, duration, and purpose; evaluated on every read.
- **Everything sensitive is audited** in an append-only, tamper-evident store.
- **Cross-tenant reads exist only as patient-consented composition, and are always audited** (tenancy invariant 6; see [10-architecture.md](10-architecture.md)).

This spec covers the authorization model and policy engine, the consent object and its lifecycle, all grant paths (QR implicit, explicit request, organization policy, integration-scoped), revocation, break-glass, the emergency lock-screen card, and the audit trail. Authentication (who the actor is) is specified in [06-identity-auth.md](06-identity-auth.md); QR token mechanics in [08-qr-subsystem.md](08-qr-subsystem.md); entity storage in [09-data-model.md](09-data-model.md); consent/audit API endpoints in [12-api-and-interoperability.md](12-api-and-interoperability.md).

---

## 2. Authorization model — the policy engine

### 2.1 Responsibilities

A central policy engine (LLD component: **Policy Engine**, interface `IAuthz`; suggested technology: OPA/Cedar-class with a consent extension, Architecture §19) evaluates **every** access to clinical data. No service reads or writes patient data without a policy decision. The decision layers three models (Architecture §7):

| Layer | Inputs | Examples |
|---|---|---|
| **RBAC** | Actor role | `patient`, `doctor`, `org-admin`, platform roles |
| **ABAC** | Attributes of actor, resource, context | tenant, facility, specialty, purpose-of-use |
| **Consent** | Active `ConsentGrant` for {patient, grantee} | categories, duration, emergency flag |

### 2.2 Evaluation contract

Every read/write carries `tenant_id + actor + purpose` (Canvas, core-services layer). The tenant context is derived server-side at the gateway and stamped as a signed, immutable header — never trusted from client input (tenancy invariant 1; see [10-architecture.md](10-architecture.md)).

```text
IAuthz.evaluate(request) -> decision

request = {
  actor:    { id, role, tenant_id, facility, specialty },
  patient:  atlas_patient_id,
  purpose:  consult | emergency | integration | self | admin,   // purpose-of-use of this access (ABAC input) — distinct from grant.purpose (§3.1), which records the grant path
  resource: { type, category, owning_tenant_id },   // omitted for slice-level evaluation
}

decision = {
  effect:            PERMIT | DENY,
  authorized_slice:  [category, ...],   // consent-trimmed categories the actor may read
  grant_ref:         grant_id | null,   // grant relied on or created (consult path)
  decision_log_id:   uuid,
}
```

Evaluation order:

1. Validate signed tenant context and authenticated session (reject otherwise — no policy evaluation on anonymous traffic).
2. RBAC gate: is this role ever allowed this operation?
3. ABAC constraints: tenant/facility/specialty/purpose rules (e.g., a doctor acts within their tenant; integrations only write, per §4.4).
4. Consent lookup: find an `active` grant for {patient, grantee}; trim the request to the granted categories.
5. Emergency policy: if `purpose = emergency`, apply the break-glass constraints of §6 instead of a patient grant.
6. Emit the decision with its inputs to the decision log (Architecture §7: "Policy decisions are cached seconds-scale and logged with inputs, for explainability").

### 2.3 The authorized slice

The set of categories a requester may read for a given patient is the **authorized slice**. It is the single authorization boundary reused everywhere:

- The records service composes the doctor's 360° view from the slice only (cross-tenant, consent-trimmed — Architecture §16 flow A step 5).
- The AI service retrieves **only from the slice**, computed per request at query time — never from a pre-built cross-record corpus (Architecture §10.2 step 1; see [05-ai-features.md](05-ai-features.md)).
- Search results expose identity only until an authorization check passes (Architecture §9 Search; see [03-doctor-app.md](03-doctor-app.md) — prototype copy under the doctor search field: "Results show identity only — records open after consent check").

### 2.4 Caching and invalidation

Policy decisions may be cached at seconds scale only. The AI summary cache is keyed by `{patient, authorized-slice hash, model version}`, so a changed consent invalidates it (Architecture §10.4). `consent.granted` / `consent.revoked` events invalidate the policy cache immediately (Architecture §16 flow C step 3).

### 2.5 Self-access and family profiles

A patient always has full read access to their own record (`purpose = self`); no grant object is involved. Guardian access to family members' profiles (Aarav and Kamala Sharma under Priya's login) flows through family/guardian links held by the identity registry — see [06-identity-auth.md](06-identity-auth.md). Proposed: guardian access is evaluated as `purpose = self` against the linked profile and appears in that profile's access log.

**Acceptance criteria**

- No clinical read or write path exists that bypasses `IAuthz.evaluate`.
- A request whose tenant context is client-supplied (unsigned) is rejected at the gateway before evaluation.
- Every decision is retrievable from the decision log with its full input set.
- Changing a grant's categories changes the authorized slice on the very next request (cache TTL ≤ seconds).

---

## 3. Consent as a first-class object

### 3.1 The ConsentGrant entity

From the LLD domain class model (see [09-data-model.md](09-data-model.md) for storage detail):

```text
ConsentGrant {
  grant_id:     UUID
  patient_id:   FK -> Patient (global atlas_id)
  grantee_id:   FK            // practitioner, organization, or integration connector
  categories:   enum[]        // mental health default-off
  duration:     24h | 30d | 90d      // MVP options exposed to the patient
  purpose:      consult | requested | org-policy | integration | emergency   // one per grant path (§4); self-access uses no grant (§2.5)
  status:       requested | active | denied | expired | revoked
  expires_at:   timestamptz
  is_emergency: boolean
}
```

Architecture §7 defines the grant as `{patient, grantee (doctor/org), categories, duration, purpose, status}`. All grants auto-expire and are revocable. The LLD class itself shows only `status: active | revoked | expired`; the `requested` and `denied` states are added here from the grant lifecycle owned by the Consent & access service — "request, approve, deny, revoke, expire" (Architecture §9).

### 3.2 Scope categories

The full category enum (Architecture §7): general history, reports & labs, medications, diagnoses, mental/behavioral health (**default off**), sensitive clinical data. The MVP patient consent UI exposes four toggles (prototype consent screen):

| Category (UI label) | Default | Notes |
|---|---|---|
| General history | On | Timeline events, consultations, hospitalizations. Proposed: the `diagnoses` enum value is grouped under this toggle in the MVP UI. |
| Reports & labs | On | Lab and imaging reports, documents |
| Medications | On | Prescriptions and medication history |
| Mental health | **Off** | Prototype helper copy: "Off by default — sensitive category". Proposed: the `sensitive clinical data` enum value receives the same off-by-default treatment when introduced. |

### 3.3 State machine

```text
                (patient approves)
  requested ──────────────────────────► active
      │                                  │  │
      │ (patient denies)   (expires_at)  │  │ (patient revokes)
      ▼                                  ▼  ▼
   denied                          expired  revoked
                                        \    /
                                     re-grant = NEW grant
                                     (original record immutable)
```

| State | Entered by | Patient-visible presentation (prototype) |
|---|---|---|
| `requested` | Doctor/org submits a consent request; patient gets a PHI-free notification | "Consent request" card with `NEW` badge on Access & consent; notification row with "Review request" action |
| `active` | Patient approves a request, or an implicit consult grant is created at QR scan (§4.1) | Green `Active` pill; "Expires in 30 days" / "Expires 12 Sep 2026"; `Revoke` action |
| `denied` | Patient denies a request. Toast: "Request denied · requester notified" | Request card disappears; no grant listed |
| `expired` | `expires_at` reached — automatic, no action needed | Proposed: grey `Expired` pill with `Re-grant` action, matching the revoked treatment |
| `revoked` | Patient revokes an active grant — immediate effect (§5) | Red `Revoked` pill; "Revoked 20 Nov 2025"; `Re-grant` action |

Terminal states are terminal: **re-granting creates a new `ConsentGrant`**; the old record is retained for audit. The QR consult path creates a grant directly in `active` (no `requested` phase — the scan itself is the consent gesture, see §4.1).

**Acceptance criteria**

- A grant's categories, duration, grantee, and purpose are immutable after creation; any change is a new grant.
- Mental health is excluded from every grant unless the patient explicitly toggled it on.
- An expired grant produces DENY without any background job having to run (expiry is checked at evaluation time against `expires_at`).
- Denying a request never reveals to the requester which categories the patient would or would not have shared.

---

## 4. Grant paths

### 4.1 QR scan — implicit consult grant

Scanning is the in-person consent gesture. Flow (Architecture §16 flow A; LLD sequence "QR consult"):

1. Patient opens the QR screen; the app fetches a fresh rotating token (TTL 300 s — see [08-qr-subsystem.md](08-qr-subsystem.md)).
2. Doctor scans; the gateway resolves the doctor's tenant, checks rate limits and the anti-replay nonce.
3. QR service resolves token → Atlas Patient ID and binds it to the doctor's session.
4. Policy engine evaluates role + consent and **creates a consult grant with default duration** — sequence message 8: `PERMIT + consultGrant(30d, categories)`.
5. Records service composes the authorized 360° slice; AI generates the clinical brief from that slice only.
6. Audit events written (scan, grant, view); patient notified "Dr. X viewed your records."

Parameters of the implicit grant:

- **Duration: 30 days** (the demo default throughout the prototype — patient access log: "12 Aug 10:40 — QR scanned · access granted (30 days)"; patient notification: "Access granted to Dr. R. Menon · 30 days · 12 Aug"). Proposed: the default duration is tenant-configurable, with 30 days as platform default.
- **Categories:** the default set — General history, Reports & labs, Medications on; Mental health off (matches the prototype's consent defaults). Proposed: patients can later trim an implicit grant's categories from the Access & consent screen.
- **Renewal by re-scan:** the doctor home shows "**Access expires** — Priya Sharma consent ends 12 Sep 2026. Re-scan her QR to renew." Re-scanning creates a fresh grant; it does not extend the old one.

Doctor-side confirmation: scan overlay copy "Identity resolved · checking consent…", then toast "Access granted · consult session logged in audit trail". The 360° header carries a session pill ("Session 28:44 · audited"); ending the consult shows "Consult ended · 1 session recorded". The consult *session* ends; the 30-day *grant* persists.

### 4.2 Explicit consent request flow

For access without an in-person scan (Architecture §16 flow C). Demo persona: Dr. Meera Kulkarni, Dermatology, SkinCare Clinic requests access to Priya's record.

**Request side.** A doctor or organization submits a consent request naming the patient. The patient receives a PHI-free notification: "Dr. Meera Kulkarni requests access — SkinCare Clinic · consent request · 8:40 today" with a **Review request** action. The Access & consent screen shows a highlighted card (1.5 px accent border): title "Consent request", `NEW` badge, body "Dr. Meera Kulkarni · Dermatology, SkinCare Clinic requests access to your records", button "Review request". Proposed: the doctor app's request-composer screen (not shown in the prototype) collects patient identifier and purpose text.

**Review screen** (patient app, "Consent request"):

1. **Requester card** — avatar initials, "Dr. Meera Kulkarni", "Dermatology · SkinCare Clinic · verified professional". The verified-professional line reflects the practitioner-registry verification of [06-identity-auth.md](06-identity-auth.md).
2. **ACCESS DURATION** — segmented pill picker: `24 hours` / `30 days` / `90 days`; default selection **30 days**.
3. **WHAT SHE CAN SEE** — the four category toggles of §3.2 with their defaults (Mental health off, with the helper copy "Off by default — sensitive category"). Toggles are live; accent-blue = on.
4. Explainer copy: "She sees only the categories you enable. Access expires automatically and every view is logged in your access log."
5. **Approve access** (primary, ink button) / **Deny** (secondary, outlined).

**Outcomes.**

- Approve → grant stored as `active` with the chosen duration and categories; policy cache invalidated; both parties notified; audit written; grant auto-expires (Architecture flow C step 3). The patient returns to Access & consent with toast "Access granted · 30 days · every view is logged" (duration text tracks the selection) and the new entry listed: "Dr. Meera Kulkarni — SkinCare Clinic · Dermatology — Active — Expires in 30 days — Revoke".
- Deny → request enters `denied`; toast "Request denied · requester notified". The requester learns only that the request was denied.

### 4.3 Organization-policy grants

Grants are also created "by organization policy for records the org itself created" (Architecture §7). Tenants own the clinical records they produce; a tenant's clinicians access those records under the org's own access policies (admin-managed — see [04-admin-app.md](04-admin-app.md) "Access policies"), without a patient-initiated grant. Patient consent governs **cross-tenant** composition: what makes the consolidated 360° view possible is the consented read-time composition across tenants, always audited (Canvas: "Why patients span tenants").

### 4.4 Integration-scoped grants

Integrations hold narrow, direction-limited grants. Prototype access list entry: "**Apollo Diagnostics** — Integration · sends reports only — Active — Ongoing — Manage".

- Scope: **write-only** ("sends reports only") — the connector may deposit finalized reports into the patient's record through the integration layer; it reads nothing. See [11-integrations.md](11-integrations.md).
- Duration: `Ongoing` — no auto-expiry, but visible in the patient's list and manageable (`Manage` action; Proposed: Manage opens a detail view with the option to stop future syncs from that source).
- Every sync is logged and appears in the patient's access log ("21 Aug 9:03 — Apollo Diagnostics · report sync").

**Acceptance criteria**

- A QR scan by an authenticated doctor with a valid token yields an `active` 30-day grant, an audit trail (scan, grant, view), and a patient notification — with no other patient action.
- A consent request never exposes clinical data to the requester before approval; the notification to the patient is PHI-free.
- Approving with Mental health off yields a slice that excludes mental-health records from the 360° view, reports list, **and** AI retrieval.
- The 24 h / 30 d / 90 d durations set `expires_at` from the approval timestamp, not the request timestamp.
- An integration grant can never be used to read patient data.

---

## 5. Revocation, expiry, and re-grant

### 5.1 Revocation — immediate effect

The patient revokes from the Access & consent screen (`Revoke`, rendered in warning red on each active grant). Effect is immediate:

- Grant status → `revoked`; `consent.revoked` event emitted; policy cache invalidated.
- The grantee's next read of any kind returns DENY. In-flight sessions lose access on their next request (Proposed: active doctor sessions for that patient are terminated server-side on revocation).
- **AI included:** retrieval is authorization-filtered at query time, so "consent revocation takes effect instantly" for summaries and Q&A (Architecture §10.2); the summary cache keyed by the authorized-slice hash is invalidated (§10.4).
- Audit event written; patient sees the entry in their log; the grantee is notified (PRD §16 "Access revoked").

### 5.2 Expiry

All grants auto-expire (Architecture §7). Expiry needs no patient action; the doctor is warned ahead of time ("Access expires — … Re-scan her QR to renew") and renewal is a new grant via re-scan or a new request. Proposed: the expiry warning to the doctor fires 30 days before `expires_at` for 30 d/90 d grants and 2 hours before for 24 h grants (matching [03-doctor-app.md](03-doctor-app.md) §5 and the prototype's doctor-home notice, shown 22 days ahead of the 12 Sep 2026 expiry).

### 5.3 Re-grant

Revoked (and, Proposed, expired) entries in the access list carry a **Re-grant** action — prototype entry: "Dr. A. Fernandes — City General · Surgery — Revoked — Revoked 20 Nov 2025 — Re-grant". Proposed behavior: Re-grant opens the consent review screen of §4.2 prefilled with the grantee and the previous grant's duration and categories; approval creates a new grant. The revoked grant record is never resurrected.

**Acceptance criteria**

- After revocation, a request made by the revoked grantee within 1 second returns DENY, and an AI question asked by them returns no content from that patient's record.
- Revocation and expiry both leave the historical grant queryable for audit.
- Re-grant produces a new `grant_id`; the audit trail shows revoke and re-grant as separate events.

---

## 6. Break-glass emergency access

For situations where the patient cannot present their QR or give consent (Architecture §16 flow D; prototype "Emergency break-glass access" screen).

### 6.1 Eligibility and entry

- Available only to authenticated clinicians (org SSO + MFA — see [06-identity-auth.md](06-identity-auth.md)). Proposed: tenants can restrict break-glass to designated roles/facilities (e.g., emergency department) via org policy.
- Entry point: the doctor QR-scan screen carries a red-tinted text link "**Emergency access — no QR available**".

### 6.2 Screen specification (doctor app)

Dark red-tinted full screen (`#1c1214`, white text) to make the exceptional mode unmistakable:

| Element | Content / behavior |
|---|---|
| Back | "‹ Back" → returns to the scan screen |
| Badge | Red circle with "!" |
| Title | "Emergency access" (display serif, 30 px) |
| Explainer | "For situations where the patient cannot present their QR or give consent. Access is limited to the emergency profile plus critical history." |
| **REASON** chips | `Unconscious patient` (selected default in the demo) · `Critical care` · `Other`. Single-select, mandatory. Proposed: choosing `Other` requires a free-text reason before Proceed enables. |
| Identifier input | Placeholder "Patient identifier (Atlas ID, ABHA or phone)" — demo value `ATL-82X92K` |
| Primary CTA | "**Proceed — logged & reviewed**" (red button) |
| Footer | "Compliance is notified immediately. Misuse ends access rights." |

Proposed error states (not depicted in the prototype): unknown identifier → inline "No patient found for this identifier"; ambiguous phone match → require Atlas ID or ABHA instead, never a pick-list of candidates.

### 6.3 Effect

On Proceed, the policy engine issues a **constrained emergency grant** (`is_emergency = true`, `purpose = emergency`):

- **Scope is limited to the emergency profile plus critical history** (Architecture §7 and flow D) — the emergency-card fields (name/age/sex, blood group, allergies with severity, conditions, current medications, emergency contact) plus critical history. Proposed: "critical history" = active diagnoses, active prescriptions, allergy records, and hospitalizations/discharge summaries from the last 12 months; excludes mental-health and sensitive categories entirely. Note: the prototype renders the standard 360° view under the banner for demo simplicity; the product must serve only the constrained slice.
- Toast on entry: "Emergency access · compliance notified · fully audited".
- The patient 360° header shows a persistent **red banner**: "**EMERGENCY ACCESS · unconscious patient**" (reason echoed) with right-aligned "compliance notified". The banner remains for the whole session and clears only when the consult ends.
- Proposed duration: the emergency grant is session-bound with a hard cap of 24 hours; it does not renew.

### 6.4 Mandatory logging, notification, and review

- Every break-glass invocation and every view inside it is audited with the reason code and identifier used (§8).
- **Compliance and the patient are alerted immediately**; the session is **flagged for post-hoc review** (Architecture flow D step 3). Patient-side copy on the emergency card: "Doctors using emergency (break-glass) access are reviewed by compliance."
- Proposed: flagged sessions surface in an admin/compliance review queue in the admin console ([04-admin-app.md](04-admin-app.md)); reviews record an outcome (justified / unjustified).
- **Misuse consequences:** "Misuse ends access rights" — Proposed mechanism: the org admin (or platform, for cross-tenant misuse) suspends the practitioner's Atlas access; suspension is itself audited.

**Acceptance criteria**

- Break-glass cannot proceed without a reason code and a patient identifier; both appear verbatim in the audit event.
- The emergency slice never includes mental-health or sensitive-category records, regardless of any prior grant.
- The compliance notification and the patient notification are emitted before the first record renders, not after the session.
- The red banner is visible on every screen of an emergency session and cannot be dismissed.
- A break-glass session appears in the patient's access log and in the compliance review queue.

---

## 7. Emergency lock-screen card (patient side)

A no-authentication counterpart for first responders: the patient's phone shows a medical card **from the lock screen — no sign-in needed** (prototype "Emergency card" screen, reached from Profile → "Emergency card · Lock-screen access").

Card content (red-headed card, exact demo values):

| Field | Value |
|---|---|
| Header | "MEDICAL EMERGENCY INFO" · right: "Atlas · ATL-82X92K" |
| Name | Priya Sharma · 34 F |
| Blood group | **B+** (bold red) |
| Allergies | **Penicillin — severe** (bold red) |
| Conditions | Type 2 diabetes |
| Medications | Metformin · Atorvastatin |
| Emergency contact | Aman S. · +91 98111 20844 |

Behavior:

- **Offline-capable:** the card (like the QR screen) is cached for offline display (Architecture §4) and continues to serve from cache if core services degrade (Architecture §18).
- **Every open is logged** — screen copy: "Anyone who opens this card is logged." Proposed: opens while offline are queued on-device and synced to the audit trail on reconnect.
- The card is read-only and contains exactly the fields above — no timeline, reports, or documents. It requires no Atlas authorization decision because the patient placed it outside the lock explicitly; enabling/disabling the lock-screen card is a patient setting (Proposed: default off until the patient turns it on during onboarding or from this screen).

**Acceptance criteria**

- The card renders with airplane mode on and the app killed.
- Opening the card writes an `emergency_card.opened` audit event (immediately when online; on next sync when offline).
- Disabling the setting removes the card from the lock screen on that device.

---

## 8. Audit trail

### 8.1 Store properties

Per PRD §13 and Architecture §9/§13: the audit store is **append-only and hash-chained**, held in WORM storage with immutable retention, **separate credentials** from application databases, hash chain **anchored daily**, and per-tenant streams with per-tenant export. Audit logs are tamper-resistant and protected from unauthorized modification; audit retention survives tenant offboarding per regulation (tenancy invariant 7).

### 8.2 Event envelope

From the LLD `AuditEvent` class and PRD §13 metadata:

```text
AuditEvent {
  audit_id:      UUID
  actor_id:      who acted        role: patient | doctor | org-admin | system | integration
  tenant_id:     FK               patient_id: FK
  action:        event type       resource:   record/document/grant reference
  access_method: qr | search | api | integration | break_glass | emergency_card
  result:        success | denied | error
  ts:            timestamptz
  hash_prev:     sha256           // chain link to the previous event in the stream
  session/request context          // per PRD §13 "relevant request/session information"
}
```

### 8.3 Event types

| Event type | Trigger | Notes |
|---|---|---|
| `qr.token_issued` | Patient app fetches a QR token | Architecture §8: issuing and resolution are both audited |
| `qr.scan_resolved` | Doctor scan resolves token → patient | PRD "Patient accessed via QR". A resolve followed by a policy DENY additionally logs the evaluation with `result = denied` (§8.2) — no separate event type |
| `qr.scan_failed` | Scan resolution fails (reason code: expired, revoked, unknown, replay) | Reason codes are audit-only; doctor-facing failures stay uniform ([08-qr-subsystem.md](08-qr-subsystem.md) §7 F1–F4) |
| `qr.scan_throttled` | Scan rate limit exceeded (per doctor or per tenant) | [08-qr-subsystem.md](08-qr-subsystem.md) §7 F5 |
| `access.viewed` | Record or document viewed | Includes 360° composition reads; PRD "Patient record viewed", "Document viewed" |
| `report.synced` | Integration deposits a report | Patient-log phrase "report sync" |
| `consent.requested` | Explicit request submitted | |
| `consent.granted` | Approval, or implicit consult grant at scan | Carries duration + categories |
| `consent.denied` | Patient denies a request | |
| `consent.revoked` | Patient revokes | |
| `consent.expired` | Grant reaches `expires_at` | Proposed: written lazily at first post-expiry evaluation |
| `break_glass.invoked` | Emergency access proceed | Carries reason code + identifier; flags session for review |
| `emergency_card.opened` | Lock-screen card opened | §7 |
| `ai.generated` | Any AI summary/brief/Q&A generation | Logs requester, slice hash, model, prompt, output (Architecture §10.5); see [05-ai-features.md](05-ai-features.md) |
| `record.created` / `record.modified` | Clinical writes (diagnosis, prescription, upload) | Doctor-app copy: "Signed as Dr. R. Menon · added to audit trail" |
| `document.uploaded` / `document.downloaded` | Document lifecycle | PRD §13 |
| `prescription.allergy_override` | Prescriber overrides an allergy conflict | Prototype toast: "Saved with allergy override — flagged to compliance" |
| `audit.exported` | Admin export run | Architecture §9: audited event classes include export ("every view, write, grant, QR scan, break-glass, export") |

Event-bus topics `consent.granted`, `consent.revoked`, `access.viewed` also drive async consumers (audit projection, notifications, AI cache invalidation — Architecture §12).

### 8.4 Patient-visible access log

The patient sees a human-readable projection on the Access & consent screen ("Recent access log" card), newest first, format `{date} {time} — {actor} · {event phrase}`:

```text
21 Aug 9:03  — Apollo Diagnostics · report sync
12 Aug 10:41 — Dr. R. Menon · viewed via QR
12 Aug 10:40 — QR scanned · access granted (30 days)
```

Screen framing copy: "Who can see your records. Every access is logged." Access events also surface as notifications ("Dr. R. Menon viewed your records — via QR scan · 12 Aug"; "Access granted to Dr. R. Menon — 30 days · 12 Aug"). The home screen summarizes active grants: "**2 providers** currently have access to your records" (count = active grants), linking to Access & consent; the profile row shows "Access & consent — 2 active ›". Proposed: the recent-log card shows the latest 3 entries with a full, filterable log one tap deeper.

### 8.5 Admin export

Org admins export their tenant's audit stream from the admin console ("Audit log — Export · 30 days"; see [04-admin-app.md](04-admin-app.md)). Export is tenant-scoped (per-tenant streams), default window 30 days. The export itself is audited (`audit.exported` — Architecture §9). Proposed: CSV and JSON formats, and asynchronous generation with notification. AI generation logs are reviewable per tenant and exportable for compliance (Architecture §10.5).

**Acceptance criteria**

- Every event type in §8.3 is written by exactly one owning service, and no code path can update or delete a written event.
- The hash chain verifies end-to-end for any exported window; a mutated event breaks verification.
- Every entry in the patient-visible log corresponds to at least one underlying audit event; the projection never invents or omits access events.
- A denied access attempt (result = `denied`) is logged just as a successful one is.
- Admin export never includes another tenant's events.

---

## 9. Access-control invariants

These bind this spec to the tenancy invariants of [10-architecture.md](10-architecture.md) (Canvas "Multi-tenancy invariants"). Each is testable and enforced at a named layer:

| # | Invariant | Enforcement point |
|---|---|---|
| 1 | The QR token contains no PHI and no identifiers; it only resolves server-side | QR token service ([08-qr-subsystem.md](08-qr-subsystem.md)) |
| 2 | `tenant_id` is derived server-side and stamped as a signed context — never trusted from the client | API gateway |
| 3 | Every clinical read/write passes the policy engine with `{tenant, actor, purpose}`; RLS is the backstop, not the primary control | Service layer + database |
| 4 | Cross-tenant reads exist **only** as patient-consented composition, and are always audited | Records service composition + audit |
| 5 | Consent is evaluated on every read; grants auto-expire; revocation is instant, including for AI retrieval | Policy engine; AI authorize-first pipeline |
| 6 | Mental-health (and future sensitive) categories are excluded unless explicitly enabled per grant | Consent category trim |
| 7 | Break-glass yields only the emergency profile + critical history, always with reason, logging, compliance + patient alerts, and post-hoc review | Emergency policy |
| 8 | Search returns identity only until an authorization check passes | Search service |
| 9 | Integrations write through the integration layer only and never read patient data | Gateway (mTLS client scope) + ABAC |
| 10 | Every sensitive operation lands in the append-only, hash-chained, per-tenant audit stream | Audit service |
| 11 | Notifications about access and consent are PHI-free | Notification templates |

**Acceptance criteria**

- Automated tests exist per invariant, including negative tests (e.g., a forged client `tenant_id` header is ignored; a cross-tenant read without a consenting grant returns DENY and an audit event).
- A penetration-test finding that violates any invariant is treated as a release blocker (PRD §17: security is a core requirement, not an enhancement).

---

## 10. Interfaces

Consent and access control surface through the versioned REST API (Architecture §15; PRD §28; detail in [12-api-and-interoperability.md](12-api-and-interoperability.md)):

```text
/consents        create request · approve · deny · revoke · list grants (patient)
/access          patient-visible access log · active-grant summary
/audit           tenant-scoped audit query + export (admin)
/qr              token issue / resolve (grant creation side effect)  — see 08-qr-subsystem.md
```

Event topics consumed/produced (Architecture §12): `consent.granted`, `consent.revoked`, `access.viewed`, `notification.requested`.
