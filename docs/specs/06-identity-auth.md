# 06 — Identity & authentication

Defines who exists in Atlas (patient, practitioner, organization, system client), how each identity is proven, and how sessions are established, maintained, and audited.

Sources: `Atlas_Project_Requirements.md` (§4, §10, §11, §12, §13, §17, §21, §22), `Atlas Architecture.dc.html` (§3–§6, §8, §9, §11, §13, §14, §16, §18, §20), `Canvas.dc.html` (layers 2–4), `Atlas LLD.dc.html` (domain class model: Patient, Practitioner, TenantOrganization, IdentityMapping, AuditEvent; QR sequence diagram), `Atlas App.dc.html` (patient welcome, OTP, QR present, profile, emergency card, family switcher; doctor login, scan, break-glass, patient 360; admin dashboard and integration queue).

Related specs: consent evaluation in [07-consent-and-access-control.md](07-consent-and-access-control.md), QR token lifecycle in [08-qr-subsystem.md](08-qr-subsystem.md), entity definitions in [09-data-model.md](09-data-model.md), gateway/tenant-context mechanics in [10-architecture.md](10-architecture.md), integration pipeline in [11-integrations.md](11-integrations.md), screen-level UI detail in [02-patient-app.md](02-patient-app.md), [03-doctor-app.md](03-doctor-app.md), [04-admin-app.md](04-admin-app.md).

## 1. Principles and scope

1. **Authentication and authorization are separate concerns** (PRD §11). This spec covers authentication (proving identity) and session management. What an authenticated actor may see is decided by the policy engine — RBAC × ABAC × consent — specified in [07-consent-and-access-control.md](07-consent-and-access-control.md).
2. **Patient identity is global; practitioner identity is tenant-scoped.** Patients are platform-level identities, not tenant members (Architecture §6). Doctors, staff, and admins exist inside exactly one tenant (healthcare organization) and authenticate through that tenant's identity provider.
3. **The QR identifies; auth decides** (Canvas layer 3). A scanned QR resolves identity only. It never grants access by itself and never contains medical data.
4. **Every authentication-adjacent sensitive event is audited** (PRD §13): sign-in, QR scan, access grant/revoke, identity-link operation, break-glass invocation.
5. `tenant_id` is derived server-side (subdomain, client certificate, or token claim) and stamped by the API gateway as a signed, immutable tenant context `{tenant_id, tier, region, plan}`. Downstream services accept only the signed header and never trust client-sent tenant IDs (Architecture §3.3, §5).

### Acceptance criteria
- No API in the platform grants data access on the basis of authentication alone; every data read/write passes the policy engine after authentication.
- A request with a client-supplied `tenant_id` that conflicts with the gateway-derived tenant context is rejected.
- Sign-in success, sign-in failure, and session termination each produce an audit event with actor, role, timestamp, access method, and result.

## 2. Identity model

### 2.1 Identity classes

| Identity class | Scope | Keyed by | Established by | Authenticates via |
|---|---|---|---|---|
| Patient | Global (platform-level, cross-tenant) | `atlas_id: UUID` + display ID `ATL-XXXXXX` | Self-registration in the patient app; or created during integration identity resolution (the no-candidate creation path is Proposed — §8) | Mobile OTP (baseline), passkey, Face ID / biometric, optional ABHA federation |
| Practitioner (doctor / staff) | Tenant-scoped | `practitioner_id: UUID`, `tenant_id: FK` | Invited/provisioned by the tenant admin via the tenant directory | Tenant IdP (OIDC/SAML) + MFA; work email + password where the tenant has no external IdP |
| Organization admin | Tenant-scoped (a practitioner-class identity with the org-admin role) | Same as practitioner | Tenant onboarding (first admin) or invitation | Same as practitioner |
| Tenant organization | Platform registry entry | `tenant_id: UUID` | Control-plane tenant onboarding (Architecture flow E) | n/a — configuration object, not a login |
| System client (hospital integration) | Tenant-scoped machine identity | Per-connector client ID | Admin integration configuration (PRD §21) | mTLS + OAuth2 client-credentials |
| Platform operator (Atlas staff) | Platform | — | Internal | JIT elevation with audit; no standing production access to PHI (Architecture §14) |

### 2.2 Atlas Patient ID

Each patient has one unique Atlas identity (PRD §4). Two identifiers exist for it:

- **Internal primary key:** `atlas_id: UUID` — marked `«global»` in the LLD class model. Used in all service-to-service references, foreign keys, and tokens. Never shown to users.
- **Display ID:** human-readable code in the format `ATL-` followed by six uppercase alphanumeric characters. Canonical example throughout the sources: `ATL-82X92K` (PRD §22; QR screen, profile, emergency card, break-glass input in the prototype). Rendered in a monospace face where it stands as an identity value (home QR card, QR present screen, profile row); it appears in the surrounding text style when inline in composed strings (emergency-card header "Atlas · ATL-82X92K", doctor 360 header "34 F · B+ · ATL-82X92K", break-glass input).

```text
Format:   ATL-XXXXXX          (X = A–Z, 0–9)
Example:  ATL-82X92K
```

Proposed: the six-character alphabet excludes visually ambiguous characters (`0/O`, `1/I/L`) so the code survives being read aloud at a reception desk; the ID is randomly assigned (non-sequential, non-derivable from phone number or demographics) and immutable for the life of the identity.

The display ID is an **identifier, not a credential**: knowing `ATL-82X92K` grants nothing. It is accepted as a lookup key in doctor patient search ("Search by name, Atlas ID or phone") and in the break-glass identifier field ("Atlas ID, ABHA or phone"), both of which are consent/policy-gated downstream.

The patient identity record (golden record) carries: name, DOB, sex, blood group, allergies, conditions, emergency contact, optional `abha_address`, and `guardian_links` (LLD Patient entity; see §7 and [09-data-model.md](09-data-model.md)).

### 2.3 Practitioner identity

From the LLD `Practitioner` entity:

```text
practitioner_id: UUID
tenant_id:       FK        — the owning organization
name · specialty
hpr_id?:         string    — professional-registry identifier (HPR-ready)
role                       — e.g. doctor, org-admin
mfa_enrolled:    boolean
```

Practitioners live in the **tenant directory**: the registry of tenants, facilities, org hierarchy, practitioner records, IdP configuration, and per-tenant feature flags (Architecture §6). A practitioner identity exists only within its tenant; the same human working at two organizations holds two practitioner identities. **Verified professional identity** (surfaced in the doctor login footer, "Verified professional identity · all activity is audited") means the practitioner record is bound to a professional-registry verification hook — HPR-ready for India deployments (Architecture §6) — via the optional `hpr_id`. PRD §11 lists professional identity verification as a future-facing requirement; the schema field and verification hook ship now, the registry integration may follow.

### 2.4 Organization (tenant) identity

From the LLD `TenantOrganization` entity: `tenant_id`, name, type, tier (`pooled | siloed | dedicated`), region (cell), `idp_config`, quotas, publication rules. Created by the control plane during tenant onboarding (Architecture flow E). The tenant's `idp_config` is what doctor/admin authentication federates against (§4).

### 2.5 System client identity

Hospital integrations authenticate as machine identities: **mTLS + OAuth2 client-credentials, scoped per tenant**, with secrets held in the platform secrets manager and never exposed to ordinary users (Architecture §6; PRD §21). Each connector carries its own credentials, identifier mappings, resource whitelists, and publication rules (Architecture §11). Detail in [11-integrations.md](11-integrations.md).

### Acceptance criteria
- Creating a patient via app registration or via integration resolution yields exactly one `atlas_id` and one display ID; the display ID matches `^ATL-[A-Z0-9]{6}$`.
- A practitioner record cannot exist without a `tenant_id`; deleting/offboarding a tenant removes its practitioner identities but never patient identities.
- Entering a valid Atlas display ID anywhere in the doctor app returns at most identity-level data until an authorization check passes.
- System-client credentials are retrievable only by the secrets manager and integration runtime, never by any user-facing API.

## 3. Patient authentication

### 3.1 Welcome / sign-in screen

Prototype screen "Patient welcome" (paper background `#faf7f1`, ink `#22303c`):

| Element | Spec |
|---|---|
| Logo | Compass-ring mark, 72 px, ocean blue `oklch(0.52 0.13 235)` |
| Headline | "Your health, in one place" — Instrument Serif, 38 px, line break after the comma |
| Subhead | "One identity, one QR, your complete medical history — shared only with doctors you allow." — 14.5 px, 60 % ink |
| Language chips | `English` (selected: ink pill, white text) · `हिंदी` · `मराठी` — language choice applies app-wide (multi-language is an approved requirement) |
| Phone input | Placeholder "Mobile number"; white field, 12 px radius; example value `+91 98220 41175` |
| Primary button | "Continue with OTP" — full-width, ink background, white text |
| Secondary action | "Sign in with Face ID" — text link, ocean blue `oklch(0.45 0.13 235)`; shown only once biometric sign-in is enrolled on this device (Proposed: hidden on first run) |
| Footer | "Works with ABHA (ABDM) · encrypted · you control access" — 12 px, 50 % ink |

Behavior: entering a mobile number and tapping "Continue with OTP" sends a one-time code by SMS and navigates to the OTP screen. Registration and sign-in share this entry point: an unknown number creates a new global patient identity after OTP verification (Proposed: followed by a minimal profile capture — name, DOB, sex — required to mint the Atlas ID).

### 3.2 OTP verification screen

Prototype screen "Patient OTP":

| Element | Spec |
|---|---|
| Back | "‹ Back" returns to welcome |
| Title | "Enter the code" — Instrument Serif, 28 px |
| Subtitle | "Sent to +91 98220 41175" — echoes the exact number the code went to |
| Code boxes | 4 boxes, 56 × 64 px, white, 12 px radius; filled boxes show the digit and take a 1.5 px ocean-blue border `oklch(0.52 0.13 235)`; the active box shows a dimmed `•` caret; empty boxes have a neutral 20 %-ink border |
| Primary button | "Verify & continue" — ink background, full width |
| Resend line | "Resend code in 0:24" — 13 px, centered, 50 % ink; live countdown (`m:ss`) |

Behavior and rules:

- The resend line counts down; the prototype captures it mid-count at `0:24`. Proposed: the cooldown starts at 0:30 per send; when it reaches zero the line becomes an active "Resend code" link, and resending restarts the timer.
- On successful verification the app navigates to patient home and (first time on a device) offers biometric enrollment (§3.3).
- Proposed (not depicted; required for a shippable flow): OTP validity 5 minutes; 5 verification attempts per code, then the code is invalidated and a new send is required; per-number and per-device send rate limits at the gateway (PRD §17 rate limiting); a wrong code shakes the boxes and shows "That code didn't match — try again"; changing the number returns to welcome via Back.
- OTP SMS content is PHI-free (notification templates carry no PHI — Architecture §9).

### 3.3 Biometric re-authentication (Face ID / platform biometrics)

- After first OTP sign-in on a device, the app offers enrollment of platform biometrics (Face ID on iOS, Class-3 biometrics on Android). Subsequent launches use "Sign in with Face ID" from the welcome screen.
- Biometric unlock **gates cached data** (Architecture §4 device posture): the offline-cached QR screen and record cache are readable only after a biometric (or OTP fallback) unlock. The one deliberate exception is the lock-screen **emergency card**, which is "Visible from the lock screen — no sign-in needed" and logs every open (prototype emergency card screen; see [02-patient-app.md](02-patient-app.md)).
- Biometric sign-in unlocks a device-bound credential (Proposed: the passkey/refresh-token private key in the platform keystore); biometrics never replace server-side session issuance.

### 3.4 Passkeys

PRD §11 lists passkeys; Architecture §6 makes them a first-class patient method ("passkeys and Face ID once enrolled"). The profile screen exposes management under **"Security & sign-in — Face ID + passkey ›"**. Spec:

- A patient may register one passkey per device (platform authenticator; WebAuthn/FIDO2). Once registered, passkey sign-in replaces OTP as the default on that device; OTP remains the universal fallback and the recovery path for new devices.
- The Security & sign-in screen (interior not depicted; Proposed) lists enrolled devices/passkeys with the ability to revoke any of them, and toggles Face ID.

### 3.5 Account recovery (Proposed)

Not depicted in the sources; required for completeness. Proposed flow, consistent with the phone-as-root-credential model:

1. **Same number, new device:** plain OTP sign-in; existing identity attaches; passkey re-enrollment offered.
2. **Lost/changed number:** "Can't access this number?" entry on welcome → verify identity via ABHA federation where linked (§6), else a manual review flow (demographic proof + cooldown) that routes to human support; the phone number on the golden record is then updated. Number changes never create a second Atlas identity — this is an identity-registry update, audited like any identity-linking operation (PRD §22).
3. All recovery events generate audit entries and a notification to the account's previous contact points.

### Acceptance criteria
- A first-time mobile number completes OTP and lands on home with a freshly minted `ATL-XXXXXX` identity; a returning number lands on its existing identity — verified by the profile screen showing the same Atlas ID across reinstalls.
- The resend link is inert while the countdown runs and functional at 0:00.
- With biometrics enrolled, cold-launching the app offline still renders the QR screen and emergency card from cache — QR only after biometric unlock, emergency card without any unlock.
- Every emergency-card open and every recovery event appears in the audit log.

## 4. Doctor authentication

### 4.1 Login screen

Prototype screen "Doctor login" — deliberately inverted palette (clinician surfaces run dark):

| Element | Spec |
|---|---|
| Background | Ink `#22303c`, white text |
| Logo | Compass-ring mark, 56 px, muted blue `oklch(0.72 0.1 235)` |
| Title | "Atlas for clinicians" — Instrument Serif, 32 px, two lines |
| Context line | "City General Hospital · SSO" — 13.5 px, 60 % white; names the resolved tenant and signals federated sign-in |
| Inputs | "Work email" (example `r.menon@citygeneral.org`), "Password" — translucent white fields on ink |
| Primary button | "Sign in with MFA" — `oklch(0.62 0.12 235)`, full width |
| Footer | "Verified professional identity · all activity is audited" — 12 px, 45 % white |

The tenant is resolved before credentials are taken (subdomain, e.g. `citygeneral.atlas.health`, or tenant picker after email-domain lookup — Architecture §5). The credential step then follows the tenant's `idp_config`: where an external IdP is configured, the button hands off to OIDC/SAML federation; where none is, Atlas-managed work email + password is used, as the prototype depicts. In both cases **MFA is enforced** — the primary button is literally "Sign in with MFA" (PRD §11: email/username + password, MFA, organization identity provider).

### 4.2 Federation and identity assurance

- **OIDC/SAML federation to the tenant's IdP, MFA enforced, professional-registry verification hook (HPR-ready)** (Architecture §6).
- The practitioner record must exist in the tenant directory before first sign-in (provisioned by the org admin — "Staff & roles" on the admin dashboard). Proposed: JIT attribute sync from IdP claims (name, specialty, role) on each sign-in, but never JIT creation of practitioner records — access is granted by the admin, not by possessing an IdP account.
- `mfa_enrolled` is tracked per practitioner; a practitioner without MFA cannot reach any clinical surface.

### 4.3 Session policy

- **Sessions are short-lived with refresh; device binding for clinical apps** (Architecture §6). Proposed concrete defaults, tunable per tenant: access token 15 min, refresh window 12 h ending in re-authentication; idle timeout 10 min in-app lock re-openable via device biometric.
- Device posture applies (Architecture §4): certificate pinning, jailbreak/root detection, no PHI in push payloads.
- Doctor **consult sessions** — the timed, per-patient window opened by a QR scan — are a distinct construct layered on the login session; see §9.3.

### Acceptance criteria
- A doctor sign-in without a satisfied MFA challenge never yields a session.
- A doctor removed from the tenant directory (or whose IdP account is disabled) fails next token refresh and loses access within the access-token lifetime.
- Sign-in from a device failing posture checks (root/jailbreak) is refused for clinical apps.
- The login screen shows the resolved organization name before credentials are entered.

## 5. Admin authentication

The organization admin (prototype: "City General Hospital · Admin" console — staff & roles, access policies, integrations, audit export) is a tenant-directory identity with the `org-admin` role and authenticates exactly as practitioners do: tenant IdP federation or work email + password, MFA enforced (PRD §3.3, §11; Architecture §4 "Admin console (web)").

- Authorization, not authentication, differentiates admins: the policy engine grants org-admin capabilities (staff management, integration credentials, error queue, publication rules, audit export) and — critically — **no clinical record access by role**. Admin visibility into the integration error queue is metadata-level ("Payload opened in secure viewer (metadata only)" — prototype toast).
- Integration credentials and secrets are configured by admins but never displayed back in full (PRD §21: "Integration credentials and secrets must never be exposed to ordinary users").
- Proposed: step-up re-authentication (fresh MFA within 5 minutes) for high-impact admin actions — viewing/rotating integration credentials, audit export, staff role changes.
- Atlas platform staff are a separate identity class: admin access to production is via JIT elevation with audit and no standing PHI access (Architecture §14); they never appear in tenant directories.

### Acceptance criteria
- An org-admin session cannot open any patient record surface; attempts are refused and audited.
- Credential values entered in integration config are write-only from the admin's perspective (masked after save).
- Audit export by an admin generates its own audit event.

## 6. ABHA / ABDM linking (India deployments)

Atlas is India-first and ABDM-aligned. ABHA integration is **optional federation at the platform-patient level** (Architecture §6: "optional ABHA (ABDM) federation in India deployments").

Surfaces fixed by the prototype:

- Welcome footer: "Works with ABHA (ABDM) · encrypted · you control access".
- Patient profile row: label "ABHA address"; value `priya@abdm` in monospace; **"Linked" badge** — 10 px, pill, background `oklch(0.93 0.05 150)`, text `oklch(0.42 0.12 150)` (the platform's green status treatment).
- Break-glass identifier field accepts ABHA as a patient identifier: placeholder "Patient identifier (Atlas ID, ABHA or phone)".

Spec:

| Aspect | Behavior |
|---|---|
| Data | `abha_address?: string` on the Patient entity (LLD) — e.g. `priya@abdm`, the ABDM `name@abdm` address format. Optional; absence changes nothing else. |
| Linking | Proposed flow (interior not depicted): from the profile ABHA row → ABDM-standard verification (ABHA OTP / face auth per ABDM specs) → on success the address is stored and the row shows the Linked badge. Unlinking from the same row, with confirmation; both operations audited as identity-link events (PRD §22). |
| Sign-in | Linked ABHA acts as a federated verification method — usable in account recovery (§3.5) and, Proposed, as an alternate sign-in. It never replaces the Atlas identity: `atlas_id` remains the primary key; ABHA is an attached external identifier. |
| Lookup | A linked ABHA address is a valid patient identifier in doctor search and break-glass, equivalent to Atlas ID or phone — identity-only until authorization passes. |
| Future | The ABDM alignment positions Atlas for national-platform federation (Architecture §20 evolution); consent-artefact interop with ABDM is out of scope for this spec. |

### Acceptance criteria
- A patient with no ABHA link sees the ABHA row empty/linkable (Proposed) and loses no functionality.
- Link and unlink operations each write an audit event naming actor, patient, and the external identifier touched.
- Entering `priya@abdm` in break-glass resolves to the same patient as `ATL-82X92K`.

## 7. Family profiles: one login, several identities

The prototype's patient home shows a **family switcher**: avatar chips "Priya · Aarav · Kamala" beneath the greeting, above the QR card; the selected chip renders inverted (ink pill, white text; unselected chips are white with an ink border). Switching changes the greeting — "Good morning, Aarav" — and scopes the whole surface to the selected member: QR, timeline, records, and consent screens all follow (the prototype simulates this and toasts "Viewing Aarav Sharma — demo data shows Priya"). Architecture: "family profiles (guardian links)" in the patient app; "family/guardian links" owned by the patient identity registry; LLD Patient carries `guardian_links: Link[]`.

Identity model:

- **Every family member is a full, separate global patient identity** — own `atlas_id`, own `ATL-XXXXXX` display ID, own QR, own consent grants, own audit trail. Aarav Sharma and Kamala Sharma are not sub-records of Priya's identity.
- A **guardian link** is a directed relationship in the identity registry: `{guardian_atlas_id, dependent_atlas_id, relationship, status}` (shape per [09-data-model.md](09-data-model.md)). It authorizes the guardian's *login* to act for the dependent's *identity*.
- The dependent identity needs no credentials of its own; it may acquire them later (Proposed: an "invite to own account" handover that adds a phone/passkey to the dependent identity and downgrades or removes the guardian link — e.g. when a child reaches majority — with both parties notified and the operation audited).

Authentication and authorization implications:

| Concern | Rule |
|---|---|
| Session | One authenticated session (the guardian's). Switching profiles is a context change, not a re-authentication. |
| Acting-as context | Every request carries both the authenticated actor and the subject patient. Audit events record actor = Priya's identity, subject = Aarav's identity (PRD §13 metadata: user, patient distinct fields). |
| Consent | Consent decisions made while switched to a dependent bind the **dependent's** grants; the notification and access log live on the dependent's record. The policy engine treats a guardian as consent-decision-maker for linked dependents. |
| QR | The QR screen while switched presents the **dependent's** token — a doctor scanning it opens the dependent's record, never the guardian's. |
| Sensitive categories | Category rules (e.g. mental health default-off) apply per subject identity, unchanged by guardianship. |
| Emergency card | Proposed: the lock-screen card shows the currently selected profile's data. |

### Acceptance criteria
- Each of Priya, Aarav, Kamala resolves to a distinct Atlas ID; scanning the QR presented while "Aarav" is selected opens Aarav's record.
- An audit query for a dependent's record shows the guardian as actor when the guardian acted.
- Revoking a guardian link immediately removes the dependent from the guardian's switcher and invalidates the guardian's ability to open the dependent's data.

## 8. Patient identity resolution during integration

External systems know their own patient IDs; Atlas must map them to global identities without ever attaching clinical data to the wrong person (PRD §22).

```text
Hospital Patient ID: HOSP-928372
        │
        ▼  Identity Resolution (crosswalk → matching → confidence)
        ▼
Atlas Patient ID: ATL-82X92K
```

Owned data — the per-tenant crosswalk (LLD `IdentityMapping`):

```text
tenant_id:   FK
external_id: string          — e.g. HOSP-928372
atlas_id:    FK              — the global identity
confidence:  float
status:      matched | pending
```

Resolution algorithm (integration pipeline step "Resolve patient identity — crosswalk HOSP-id → ATL-id · confidence scoring", LLD activity flowchart; Architecture §9 identity registry):

1. **Deterministic:** exact hit on the tenant's crosswalk (`{tenant_id, external_id}` → `atlas_id`, status `matched`) resolves immediately. This is the steady-state path.
2. **Probabilistic:** no crosswalk hit → match demographics (name, DOB, sex, phone, and ABHA where present — "appropriate matching attributes", PRD §22) against golden records with confidence scoring.
   - Confidence ≥ auto-match threshold with a single candidate → create the crosswalk entry (`matched`) and proceed. Duplicate-prevention: an existing patient who can be confidently identified must not yield a new Atlas record (PRD §22).
   - Below threshold with no plausible candidate → Proposed: create a new global identity plus crosswalk entry, flagged for later merge review.
   - **Ambiguous (multiple candidates, or borderline confidence) → route to the human resolution queue. Never auto-attach.** The LLD renders this as an explicit constraint node: "«constraint» never auto-attach to wrong patient". The mapping is stored with status `pending` and the clinical event holds in the error queue.
3. **Every identity-linking operation is audited** (PRD §22) — crosswalk creation, manual resolution, merges.

Admin-facing behavior (prototype "Legacy HIS — events" queue): a failed event reads "Lab result LAB-102993 — `HOSP-928372 → ambiguous identity (2 candidate matches)`" with a **"Resolve identity"** action; the confirmation copy is "Routed to identity resolution — never auto-attached". Resolution UI detail in [04-admin-app.md](04-admin-app.md); queue states and retries in [11-integrations.md](11-integrations.md).

| Outcome | Crosswalk status | Clinical event | Audit |
|---|---|---|---|
| Deterministic hit | `matched` (existing) | Processed | Link reuse logged with event provenance |
| Confident single probabilistic match | `matched` (new entry) | Processed | New link logged, confidence recorded |
| No candidate | `matched` to new identity (Proposed) | Processed | New-identity creation logged |
| Ambiguous | `pending` | Held in admin resolution queue (FAILED, never dropped) | Routing logged; manual decision logged with resolver identity |

### Acceptance criteria
- Replaying the same `{tenant, source_system, source_record_id, version}` never creates a second crosswalk entry or duplicate patient (pipeline idempotency key, Architecture §11.2).
- An ambiguous match produces zero writes to any patient's clinical record until a human resolves it.
- Resolving a pending mapping releases the held clinical event through the normal pipeline and both actions appear in the audit log.
- Matching thresholds are configuration, reviewable per deployment (Proposed).

## 9. Session management

### 9.1 Session types

| Session | Holder | Established by | Lifetime | Bound to | Audited |
|---|---|---|---|---|---|
| Patient app session | Patient (guardian may act for dependents) | OTP / passkey / Face ID | Long-lived refresh; biometric gate on cached data (Proposed: refresh 30 d rolling) | Device | Sign-in/out events |
| Doctor login session | Practitioner | Tenant IdP + MFA | Short-lived access + refresh (Architecture §6; Proposed defaults §4.3) | Device (device binding) | Sign-in/out, refresh failures |
| **Doctor consult session** | Practitioner × one patient | QR scan resolution (or break-glass) | Timed, minutes-scale; prototype shows `Session 28:44 · audited` | Doctor's login session + patient + consult grant | Every view within it |
| Admin console session | Org admin | Tenant IdP + MFA | As doctor login session; Proposed step-up for sensitive actions | Browser session | All admin actions |
| System client session | Integration connector | mTLS + OAuth2 client-credentials | Token-lifetime per OAuth2; per-tenant scope | Client cert + tenant | Source-authentication step logged per event |

### 9.2 Patient sessions

- Persistent sign-in is the norm; the cost of losing a phone is bounded by the biometric gate on cached data and by server-side session revocation (Proposed: "sign out other devices" under Security & sign-in).
- Offline behavior: the QR screen and the emergency card are cached for offline display (Architecture §4); QR presentation and the emergency card serve from cache if core services degrade (Architecture §18). A cached QR token is only as fresh as its last rotation — see [08-qr-subsystem.md](08-qr-subsystem.md) for offline-token validity rules.

### 9.3 Doctor consult sessions (timed, audited)

The QR consult flow (Architecture flow A; LLD sequence diagram) creates a **consult session** distinct from the doctor's login session:

1. QR service resolves token → Atlas Patient ID and **binds the token to the scanning doctor's session** (session binding, anti-replay nonce consumed).
2. Policy engine evaluates role + consent; a **consult grant is created with default duration** ([07-consent-and-access-control.md](07-consent-and-access-control.md)).
3. The patient-360 surface opens under a visible countdown pill: **"Session 28:44 · audited"** (11 px, pill, background `oklch(0.94 0.02 235)`), counting down live. The demo value implies a ~30-minute default consult window (Proposed: 30 minutes, tenant-configurable).
4. Every view inside the session is audited (scan, grant, view events — flow A step 6) and the patient is notified "Dr. X viewed your records."
5. Expiry: when the timer reaches zero the record surface closes/locks (Proposed: a "session ended — re-scan to continue" interstitial; unsaved clinical drafts are preserved). Re-scanning the patient's QR opens a fresh session. Longer-term access is the consent grant's business, not the session's: the doctor-home notice "Access expires — Priya Sharma consent ends 12 Sep 2026. Re-scan her QR to renew" shows grant-level expiry handled separately.
6. Break-glass opens a consult session the same way but under an emergency grant, with a red banner replacing the normal consent basis — left "EMERGENCY ACCESS · unconscious patient", right "compliance notified" (prototype); it is flagged for post-hoc review (Architecture flow D).

Consult sessions are stored in the tenant-namespaced Redis-class session/QR-state store, TTL-bound, no durable PHI (Architecture §13).

### Acceptance criteria
- A consult session's remaining time is always visible on the patient-360 header and reaches zero exactly at server-side expiry (server is authoritative; the pill is display).
- After consult-session expiry, any further record read for that patient requires a new scan/grant and is otherwise refused.
- Revoking a doctor's login session (IdP disable, admin action) kills its live consult sessions within the access-token lifetime.
- Every consult session maps 1:1 to an auditable chain: scan → grant → views → expiry/termination.

## 10. Tokens and rotation

Authoritative detail lives in [08-qr-subsystem.md](08-qr-subsystem.md); this section fixes the boundary.

| Token | Format | Lifetime / rotation | Notes |
|---|---|---|---|
| QR token | Opaque, resolvable only by Atlas — **no identifiers, no PHI** (PRD §10; Architecture §8) | Short-lived (minutes), rotates automatically — prototype pill: "Secure token · rotates in 4:52" (≈5-minute rotation); revocable | Session binding, nonce anti-replay, per-doctor and per-tenant rate limits; stricter gateway anti-replay on QR endpoints. On-screen reassurance copy: "This code identifies you — it contains no medical data." |
| Static/printed fallback QR | Opaque, long-lived | Revocable | Resolves into a higher-friction flow requiring an additional identity check (Architecture §8) |
| Access / refresh tokens | Proposed: JWT access + opaque rotating refresh | Access short-lived; refresh rotated on use (Proposed) | Carry the claims the gateway may use for tenant resolution; never carry PHI |
| Tenant context | Signed header `{tenant_id, tier, region, plan}` | Per-request | Stamped by the gateway; the only tenant assertion services accept |
| Integration client tokens | OAuth2 client-credentials over mTLS | Per OAuth2 token lifetime | Secrets in platform secrets manager; rotation via admin integration config |

QR state lives in a replicated cache with strict TTL; issuing and resolution are both audited (Architecture §8).

### Acceptance criteria
- Decoding any Atlas QR yields no name, phone, Atlas ID, or clinical data.
- A captured QR token replayed after its rotation window, or by a session other than the one it was bound to, is refused and the attempt audited.
- No token class in the table embeds PHI.

## 11. Cross-cutting security requirements

From PRD §17 as they bear on this spec: strong authentication, secure session management, secure QR token design, rate limiting, replay protection, comprehensive audit logging, secrets management — all mandatory, not enhancements. Defense-in-depth placement (Architecture §14): the gateway performs tenant resolution, rate limiting, and anti-replay; services perform authN + policy on every call with purpose-of-use recorded; audit is append-only and hash-chained (`hash_prev: sha256` on `AuditEvent`) with WORM retention. Compliance targets (DPDP, ABDM alignment, HIPAA/GDPR where applicable — PRD §18) are tracked in [13-non-functional.md](13-non-functional.md).

### Acceptance criteria
- Penetration testing of the auth surface (OTP flooding, OTP brute force, QR replay, session fixation, tenant-header forgery) shows each attack blocked at the layer named above.
- Audit chain verification detects any mutation of an authentication-related audit record.
