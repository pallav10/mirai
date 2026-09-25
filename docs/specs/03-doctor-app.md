# 03 — Doctor App

Implementation spec for the Atlas clinician-facing mobile app: authentication, patient lookup, QR-initiated consults, emergency access, the patient 360° view with AI clinical brief, clinical write flows, and the consult session model.

Sources: `Atlas App.dc.html` (doctor screens and logic script), `Atlas_Project_Requirements.md` (§3.2, §6–8, §10, §12–17, §24, §33), `Atlas Architecture.dc.html` (§4 clients, §6 identity, §7 authorization, §8 QR, §9 services, §10 AI, §13 data/session stores, §16 flows A/C/D, §17 SLOs), `Atlas LLD.dc.html` (QR-consult sequence, ConsentGrant model), `Atlas Directions.dc.html` (direction 1a Meridian).

---

## 1. Scope and role

The doctor app is used by verified healthcare professionals who belong to a tenant organization (e.g. City General Hospital). Doctors retrieve a patient's consolidated record during a consultation and add new clinical information to it (PRD §3.2). The app supports iOS and Android from a single design (device frames in `ios-frame.jsx` / `android-frame.jsx`); layout, copy, and colors below are identical on both platforms.

Everything in this app operates inside the security frame set by PRD §33: **the QR identifies the patient; the doctor's authentication and authorization decide what they see.** Every read and write is audited (PRD §13), and the patient can see the resulting access events in their own app (see [02-patient-app.md](02-patient-app.md), notifications and access log).

Demo persona used throughout: Dr. Rohan Menon, Endocrinology, City General Hospital, consulting patient Priya Sharma (34 F, B+, ATL-82X92K, severe penicillin allergy).

---

## 2. Navigation model

```text
Login (org SSO + MFA, dark)
  └─▶ Home
        ├─▶ QR scan (dark) ──▶ [resolve + consent check] ──▶ Patient 360
        │        └─▶ Break-glass emergency access (dark red) ──▶ Patient 360 (emergency)
        ├─▶ Patient search results (identity only) ──▶ Patient 360 (after consent check)
        └─▶ Recent patient ──▶ Patient 360 (active grant required)
Patient 360 (tabs: Overview · Timeline · Reports · Meds)
        ├─▶ + Add to record (Diagnosis | Prescription | Report) ──save──▶ 360 · Timeline tab
        └─▶ End consult ──▶ Home (session closed, state reset)
```

| Screen | Route id (proposed) | Entry points | Exits |
|---|---|---|---|
| Login | `doctor/login` | App launch, session expiry | Home (successful MFA) |
| Home | `doctor/home` | Login, End consult, Cancel scan, back from 360 | Scan, Patient 360, search |
| QR scan | `doctor/scan` | Home scan CTA | Home (Cancel), Patient 360 (scan resolved), Break-glass |
| Break-glass | `doctor/emergency` | Scan screen link | Scan (Back), Patient 360 with emergency banner (Proceed) |
| Patient 360 | `doctor/patient/:atlasId` | Scan, Break-glass, recent patient, search result | Home (`‹ Patients`, End consult), Add to record |
| Add to record | `doctor/patient/:atlasId/add` | 360 action bar | 360 (back link, or save → Timeline tab) |

The device status bar/frame renders **dark** on Login, QR scan, and Break-glass; all other screens render on the standard warm paper background `#faf7f1` with ink `#22303c` (see [01-design-system.md](01-design-system.md) for the full token set).

**Acceptance criteria**

- An unauthenticated user can reach no screen other than Login.
- Every path into Patient 360 passes an authorization check (grant exists, scan resolution, or emergency grant); there is no route that renders clinical data without one.
- End consult always returns to Home and clears consult-scoped state (§10).

---

## 3. Shared conventions

- **Typography:** Instrument Serif for display headings (screen titles, patient name, section headers like "Recent patients"); Instrument Sans for all body/UI text.
- **Toasts:** confirmation/audit toasts render as a dark `#22303c` rounded (12px) bar pinned 36px above the bottom edge, white 13px text, slide-up entrance ~250ms, auto-dismiss after ~2.8s. Exact toast copy is specified per interaction below. The prototype exposes a `showAuditToasts` switch; in product these toasts always show — they are the doctor-facing trace that actions were audited.
- **Status pills** (used on reports and meds): Final/Active = green (`oklch(0.93 0.05 150)` bg, `oklch(0.42 0.12 150)` text); Preliminary = amber (`oklch(0.93 0.06 80)` bg, `oklch(0.45 0.12 60)` text); Amended = blue (`oklch(0.94 0.02 235)` bg, `oklch(0.42 0.12 235)` text); Done/other = neutral (`rgba(34,48,60,0.08)` bg, `rgba(34,48,60,0.6)` text).
- **Device posture** (architecture §4): certificate pinning, jailbreak/root detection, no PHI in push payloads, biometric unlock gating any cached data.

---

## 4. Login (org SSO + MFA)

Dark full-bleed screen: background `#22303c`, white text, content vertically centered, 28px horizontal padding.

**Layout, top to bottom**

1. Compass-ring logo, 56px, stroke `oklch(0.72 0.1 235)` (the light-on-dark accent variant).
2. Display title, Instrument Serif 32px, two lines: `Atlas for clinicians`.
3. Org context line, 13.5px, `rgba(255,255,255,0.6)`: `City General Hospital · SSO`. This names the tenant the doctor is signing into.
4. Input `Work email` (demo value `r.menon@citygeneral.org`) — 12px radius, `rgba(255,255,255,0.08)` fill, `rgba(255,255,255,0.25)` border, white text.
5. Input `Password` (masked).
6. Primary button, full width, `oklch(0.62 0.12 235)` background, white 15px semibold: `Sign in with MFA`.
7. Footer note, centered, 12px, `rgba(255,255,255,0.45)`: `Verified professional identity · all activity is audited`.

**Behavior**

- Tapping `Sign in with MFA` completes authentication and lands on Home. The prototype collapses the flow to one tap; the product flow is: credentials → tenant IdP federation (OIDC/SAML per architecture §6) → MFA challenge (enforced, method per org IdP) → short-lived session with refresh token and device binding. See [06-identity-auth.md](06-identity-auth.md) for the token contract and professional-registry (HPR-ready) verification hook.
- The "verified professional identity" note reflects a real state: only doctors whose practitioner record is verified in the tenant directory may sign in. Proposed: an unverified account that authenticates at the IdP sees a blocking "Your professional identity is pending verification by your organization" state instead of Home.
- Proposed error states (not in prototype): invalid credentials inline error under the password field; MFA failure returns to the challenge with a retry; IdP unreachable shows a non-blocking banner with retry.

**Acceptance criteria**

- A doctor cannot obtain a session without completing the org MFA challenge.
- Sessions are short-lived and refresh silently; an expired refresh returns the user to this screen.
- Successful sign-in emits an audit event (login, actor, tenant, device).

---

## 5. Home

Standard light screen, scrollable.

**Header** — left: eyebrow 13px `rgba(34,48,60,0.55)` `City General · Endocrinology`, below it the doctor's name in Instrument Serif 26px (`Dr. Rohan Menon`). Right: 40px circular avatar with initials (`RM`), `oklch(0.9 0.03 235)` background, `oklch(0.42 0.12 235)` text.

**Scan CTA card** — dark `#22303c` card, 16px radius, 18px padding, full width (22px margins). Left icon: 46px rounded square outlined 2.5px in `oklch(0.72 0.1 235)` with a filled QR-target square inside. Copy: title 16px semibold `Scan patient QR`; subtitle 12.5px `rgba(255,255,255,0.6)` `Start a consult — identity & consent checked automatically`. Tap → QR scan screen. This is the primary action on the screen.

**Patient search** — full-width input, placeholder `Search by name, Atlas ID or phone`. Helper line beneath, 11px `rgba(34,48,60,0.45)`: `Results show identity only — records open after consent check`.

- Search filters as the user types (live, per keystroke in the prototype; product: debounced query to the search service).
- **Identity-only results rule** (PRD §15, architecture §9 Search): result rows may show only identity data — name, initials avatar, and demographic/context meta line. No clinical detail may be fetched or rendered for a result until an authorization check passes when the row is opened. (The prototype's meta lines include a condition because its roster doubles as "recent patients" the doctor has already treated — see below; pure search results for patients with no relationship must show identity fields only. Proposed meta for such rows: age, sex, city/masked phone.)
- Opening a result runs the consent check: if an active grant exists the Patient 360 opens; if not, the app must not show records. Proposed (consistent with the helper copy and flow C in architecture §16): the app offers "Request access" which sends the patient a consent request (see [07-consent-and-access-control.md](07-consent-and-access-control.md)), or the doctor scans the patient's QR instead.
- **Org-specific identifier (deferred).** PRD §15 lists the organization-specific patient identifier (hospital MRN) as a potential search key, resolvable through the tenant's IdentityMapping crosswalk ([09-data-model.md](09-data-model.md) §12). Proposed disposition: deferred post-MVP — MVP search keys are name, Atlas ID, and phone (this screen), plus DOB at the API ([12-api-and-interoperability.md](12-api-and-interoperability.md) §3.2); MRN lookup would return identity-only results like any other key and requires no model change when added.

**Recent patients** — section title Instrument Serif 19px `Recent patients`, then a card list (14px radius, white, 1px `rgba(34,48,60,0.1)` border). Row anatomy: 38px initials avatar, name 14px semibold, meta 12px `rgba(34,48,60,0.55)`, trailing chevron `›`. Demo roster:

| Name | Meta line | Opens |
|---|---|---|
| Priya Sharma | `34 F · Type 2 diabetes · seen 12 Aug` | Patient 360 (active grant to 12 Sep 2026) |
| Arjun Rao | `58 M · hypertension · seen 11 Aug` | 360 subject to grant check |
| Meena Joshi | `42 F · asthma · seen 08 Aug` | 360 subject to grant check |
| Sanjay Patel | `61 M · CKD stage 2 · seen 05 Aug` | 360 subject to grant check |

Recent patients are patients this doctor has an existing treatment relationship with; their meta may include the primary condition because the doctor is already authorized for those records. Search matches against both name and meta text. (In the prototype only Priya's record is populated; tapping the others shows the toast `Prototype: only Priya's record is populated` — product behavior is identical to opening any authorized patient.)

- Opening a recent patient with a still-active grant goes straight to the 360. The AI brief renders from cache without the generating skeleton (it was generated at the last scan and the summary cache is keyed by patient + authorized slice + model version, architecture §10.4); a changed record or changed consent regenerates it.
- Opening a recent patient whose grant has expired must not show records. Proposed: the row opens a blocked state with "Consent expired — re-scan QR or request access."

**Consent-expiry notice** — informational card at the bottom, `oklch(0.94 0.02 235)` background, 14px radius, 12px/16px padding, 12px text at `rgba(34,48,60,0.7)`: `**Access expires** — Priya Sharma consent ends 12 Sep 2026. Re-scan her QR to renew.` Shown when any of the doctor's active grants expires within a warning window (Proposed: 30 days). Re-scanning creates a fresh consult grant (§10).

**Acceptance criteria**

- Search never returns clinical fields for a patient the doctor holds no active grant for.
- A recent-patient row for an expired grant does not open the 360.
- The expiry notice names the patient and exact end date of the earliest-expiring active grant.

---

## 6. QR scan

Dark camera screen: background `#101820`, white text, 28px padding, dark device frame.

**Layout**

1. Top-left link `‹ Cancel` (14px, `rgba(255,255,255,0.7)`) → Home.
2. Centered viewfinder: 240×240px area, 24px-radius faint fill `rgba(255,255,255,0.04)`, four 44px corner brackets stroked 3.5px in `oklch(0.72 0.1 235)` with 18px corner radii.
3. Instruction, centered 14px `rgba(255,255,255,0.7)`, max-width 250px: `Align the patient's Atlas QR within the frame`.
4. Pill button `Simulate scan` (prototype affordance — in product the camera detects the code automatically; no button).
5. Text link in warning red `oklch(0.75 0.12 25)` with underline: `Emergency access — no QR available` → Break-glass screen (§7).
6. Footer, centered 12px `rgba(255,255,255,0.45)`: `The QR identifies the patient — your authorization decides what you see.`

**Scan behavior (product)**

The scan drives architecture flow A / LLD sequence "QR consult":

1. Camera detects the patient's Atlas QR (an opaque, short-lived rotating token — no PHI; see [08-qr-subsystem.md](08-qr-subsystem.md)).
2. App calls `POST /qr/resolve {token, nonce}`. The gateway checks tenant context, rate limits, and the anti-replay nonce; the QR service resolves the token to an Atlas Patient ID and binds it to this doctor's session.
3. **In-viewfinder progress state:** the recognized code renders as a small white card (~130px) inside the frame with the caption `Identity resolved · checking consent…` in `oklch(0.8 0.1 235)`. In the prototype this state lasts ~1.1s.
4. The policy engine evaluates role + consent; on PERMIT a **consult grant** is created with the default duration and categories (30 days in the LLD sequence; category defaults per [07-consent-and-access-control.md](07-consent-and-access-control.md) — mental-health category off unless explicitly granted).
5. The app navigates to Patient 360, Overview tab, and shows the toast: `Access granted · consult session logged in audit trail`. The AI clinical brief starts generating at this moment and shows its skeleton for ~1.6s in the prototype (product SLO: p95 < 2.5s, architecture §10.4).
6. Async: audit events written (scan, grant, view) and the patient receives a PHI-free notification ("Dr. R. Menon viewed your records · via QR scan" appears in the patient app).

**Failure states** (grounded in architecture §8; presentation Proposed since the prototype only shows the happy path)

| Condition | Behavior |
|---|---|
| Token expired/rotated | Inline message in viewfinder: ask the patient to refresh their QR screen; resume scanning. |
| Token replayed / nonce consumed | Treated as expired; audit event recorded. |
| Rate limit exceeded | Blocking message with retry-after; audited. |
| Policy DENY (e.g. patient revoked all access) | Identity was resolved, so render the identity-only card — name, age/sex, Atlas ID, mirroring search's identity-only results — with the message "Access not authorized — ask the patient to approve a consent request" and a "Request access" action starting the consent-request flow ([08-qr-subsystem.md](08-qr-subsystem.md) §7 F6). No clinical data shown; denial audited. |
| Offline / gateway unreachable | Scan cannot proceed; show connectivity error. QR resolution is server-side only — the app never decodes patient identity locally. |

**Acceptance criteria**

- The QR payload is never parsed for identity or PHI on-device; resolution happens only via the API.
- A successful scan produces exactly one consult grant and one audit trail entry set (scan + grant + view).
- The patient is notified of the access without PHI in the push payload.
- Cancel returns to Home without any resolution call side effects.

---

## 7. Break-glass emergency access

Dark red-tinted screen: background `#1c1214`, white text, dark frame. Reached only from the scan screen's `Emergency access — no QR available` link.

**Layout**

1. Top-left `‹ Back` → QR scan.
2. Alert badge: 46px circle, `oklch(0.55 0.18 25)`, bold white `!`.
3. Title, Instrument Serif 30px, two lines: `Emergency access`.
4. Body, 13.5px `rgba(255,255,255,0.65)`: `For situations where the patient cannot present their QR or give consent. Access is limited to the emergency profile plus critical history.`
5. Label `REASON` (11.5px, letterspaced, `rgba(255,255,255,0.5)`) above a wrapping row of single-select reason chips: `Unconscious patient` (selected in demo — filled `oklch(0.55 0.18 25)`), `Critical care`, `Other` (unselected — `rgba(255,255,255,0.1)` fill, `rgba(255,255,255,0.2)` border). A reason is **mandatory** (architecture §7, flow D). Proposed: selecting `Other` requires a free-text reason.
6. Input, placeholder `Patient identifier (Atlas ID, ABHA or phone)` — demo value `ATL-82X92K`.
7. Primary button, `oklch(0.55 0.18 25)`: `Proceed — logged & reviewed`.
8. Footer, centered 12px `rgba(255,255,255,0.5)`: `Compliance is notified immediately. Misuse ends access rights.`

**Proceed behavior**

1. The policy engine issues a **constrained emergency grant** scoped to the emergency profile plus critical history (architecture §16 flow D). The grant is recorded with `is_emergency = true` (LLD ConsentGrant).
2. Compliance and the patient are alerted immediately; the session is flagged in audit for post-hoc review.
3. The app opens Patient 360, Overview tab, with the toast `Emergency access · compliance notified · fully audited`, and the AI brief generates as in a normal scan (~1.6s skeleton).
4. A persistent **red emergency banner** renders in the 360 header (below the identity row, above the allergy banner): `oklch(0.55 0.18 25)` background, white 12px semibold, left text `EMERGENCY ACCESS · unconscious patient` (the chosen reason, lowercased), right text at 85% opacity `compliance notified`. The banner remains for the whole emergency session and is cleared by End consult.

**Scope limitation.** Under an emergency grant the 360 composes only the emergency profile (blood group, allergies, existing conditions, active medications, emergency contact) plus critical history as defined in [07-consent-and-access-control.md](07-consent-and-access-control.md) §6.3 — active diagnoses, active prescriptions, allergy records, and hospitalizations/discharge summaries from the last 12 months; mental-health and sensitive categories are excluded entirely. The prototype renders Priya's full demo record; production must trim the slice server-side to the emergency scope. Proposed rendering of trimmed areas: tabs remain visible but sections outside scope show "Not available under emergency access." Write actions (`+ Add to record`) remain available — treatment given in an emergency must still be recordable; each write is flagged to the emergency session in audit (Proposed).

**Acceptance criteria**

- Proceed is disabled until a reason is selected and a patient identifier is entered.
- The emergency grant never includes consent-gated categories (e.g. mental health) or the full timeline; the served slice is limited server-side, not by UI hiding.
- Compliance notification and patient notification fire on grant creation, not on first record view.
- The emergency banner is visible on every tab of the 360 for the entire session and cannot be dismissed manually.

---

## 8. Patient 360

The consolidated consult view (PRD §14). Structure: fixed header (identity + banners + tabs), scrollable tab body, fixed bottom action bar.

### 8.1 Header

Background `#fdfbf7`, bottom hairline `rgba(34,48,60,0.1)`.

- **Top row:** left link `‹ Patients` (→ Home; does not end the session — Proposed: the session keeps running and the patient reopens from Recent patients until End consult or timeout). Right: session pill — `oklch(0.94 0.02 235)` background, 999px radius, 11px `rgba(34,48,60,0.5)`: `Session 28:44 · audited`. The time is a live countdown of the remaining consult session (§10).
- **Identity row:** 46px initials avatar (`PS`, `oklch(0.9 0.03 235)` / `oklch(0.42 0.12 235)`), patient name Instrument Serif 22px (`Priya Sharma`), meta 12px `rgba(34,48,60,0.55)`: `34 F · B+ · ATL-82X92K` (age, sex, blood group, Atlas ID).
- **Emergency banner** (emergency sessions only, §7).
- **Allergy banner — always visible on every tab:** `oklch(0.95 0.03 25)` background, 1px `oklch(0.85 0.06 25)` border, 10px radius, 12.5px text in `oklch(0.42 0.15 25)`: `**Allergy:** Penicillin — severe reaction (2019)`. Rendered whenever the patient has any recorded allergy; lists the allergy, severity qualifier, and year. Proposed: multiple allergies render as one banner listing each, most severe first.
- **Tab strip:** `Overview · Timeline · Reports · Meds` — 12.5px semibold; active tab in ink `#22303c` with a 2px underline in accent `oklch(0.45 0.13 235)`; inactive `rgba(34,48,60,0.5)`. Default tab after any entry: Overview.

### 8.2 Overview tab

A vertical stack of cards (10px gap, 22px side margins).

**AI clinical summary card** — visually distinct: gradient background `linear-gradient(135deg, oklch(0.96 0.02 235), oklch(0.94 0.03 260))`, 1px border `oklch(0.86 0.05 245)`, 14px radius.

- Header row: small rotated-diamond glyph in `oklch(0.5 0.14 260)`; label 11px, letterspaced, `oklch(0.42 0.12 260)`: `AI CLINICAL SUMMARY`; while generating, a right-aligned 10.5px `Generating…`.
- **Generating state:** three skeleton bars (9px tall, `oklch(0.88 0.03 250)`, widths ~95% / 85% / 60%). Shown for the generation window after a scan or emergency entry (~1.6s in the prototype; p95 < 2.5s SLO). When the brief is served from cache (reopening an authorized patient), the ready state renders immediately with no skeleton.
- **Ready state** — brief text, 13px, line-height 1.55. Exact demo copy (bold spans as marked):

  > 34 F, T2DM (2022), well controlled — HbA1c 6.8% (Jun). New lipid panel today: **TG 168 ↑**, LDL 104, otherwise in range. On Metformin 500 BD + Atorvastatin 10 OD since Aug. Appendectomy Nov 2025, uneventful. **Penicillin allergy.** Suggested focus: hypertriglyceridemia — diet review vs. dose adjustment.

  The brief always covers: demographics + conditions, latest results with abnormal flags (the TG flag), current medications, relevant history, allergies, and a **suggested focus** line. It is generated only from the doctor's authorized slice, with claim verification against source records (architecture §10.2); see [05-ai-features.md](05-ai-features.md) for the pipeline contract.
- **Provenance + disclaimer line**, 10.5px `rgba(34,48,60,0.45)`: `Generated from 7 records at scan · verify against source data before clinical decisions`. The record count is real (size of the retrieved set). This disclaimer is mandatory and non-configurable.
- **Ask-the-record chips** — wrapping row of suggested question chips: 11.5px semibold, `rgba(255,255,255,0.75)` fill, `oklch(0.86 0.05 245)` border, `oklch(0.42 0.12 260)` text, 999px radius. Demo chips and their exact answers:

| Chip | Inline answer |
|---|---|
| `Last eye exam?` | `No ophthalmology records in Atlas. Retinopathy screening was advised at diagnosis (Mar 2022) — no result on file. Consider referral.` |
| `HbA1c trend?` | `7.4% (Dec 2024) → 7.0% (Mar 2025) → 6.8% (Jun 2026). Improving; under 7% target for 14 months.` |
| `Med adherence?` | `Metformin refills on schedule for 12 months. Atorvastatin started 12 Aug — first refill due 11 Sep.` |

- Tapping a chip renders one **inline answer card** inside the summary card: `rgba(255,255,255,0.8)` fill, 10px radius; question in 11px bold `oklch(0.42 0.12 260)`, answer 12.5px. Only one answer shows at a time — tapping another chip replaces it. Answers are generated strictly from the record with the same verify-and-cite pipeline; the open answer is cleared on End consult. Note the first answer demonstrates the honest-absence behavior: the AI states what is *not* in the record rather than guessing. Proposed: free-text follow-up entry in addition to chips is a [05-ai-features.md](05-ai-features.md) concern; the prototype ships chips only.
- AI output is never written to the record and triggers no clinical action (architecture §10.5).

**CONDITIONS card** — white card, label style 11px semibold letterspaced `rgba(34,48,60,0.5)`. Condition chips (`oklch(0.94 0.02 235)` fill, `oklch(0.4 0.11 235)` text): `Type 2 diabetes · 2022`, `Allergic rhinitis`.

**CURRENT MEDICATIONS card** — 13.5px, line-height 1.7: `Metformin 500 mg — twice daily` / `Atorvastatin 10 mg — once daily, night`.

**LATEST RESULTS card** — two rows, label left / value right (13.5px): `Lipid panel · today` → `TG 168 ▲` in amber `oklch(0.55 0.15 60)` semibold (abnormal flag); `HbA1c · 03 Jun` → `6.8%` semibold. Abnormal values always render flagged in amber with the ▲/▼ marker.

**LAST VISIT card** — 13.5px: `12 Aug — follow-up, Type 2 diabetes. Renewed Metformin; advised lipid panel. (Dr. R. Menon)` with the attribution in `rgba(34,48,60,0.55)`.

### 8.3 Timeline tab

The same year-grouped chronological timeline the patient sees in their own app (see [02-patient-app.md](02-patient-app.md)); the doctor view has **no filter pills** — it always shows all event types, trimmed to the authorized slice.

- **Group header:** year label 11px semibold letterspaced `rgba(34,48,60,0.45)` with a hairline rule filling the row.
- **Event card:** white, 14px radius; left 38px rounded tint chip carrying a 2–3 letter type code; then date · type line (11px `rgba(34,48,60,0.5)`), title (14px semibold), organization (12px `rgba(34,48,60,0.55)`); trailing `PDF` badge (`oklch(0.94 0.02 235)` / `oklch(0.42 0.12 235)`) when a document is attached. Chip tint: background `oklch(0.93 0.045 H)`, text `oklch(0.4 0.12 H)` with a hue per type.

| Type code | Event type | Hue H |
|---|---|---|
| LAB | Lab report | 235 |
| CON | Consultation | 280 |
| RX | Prescription | 150 |
| VAX | Vaccination | 190 |
| HSP | Hospitalization | 60 |
| DX | Diagnosis | 25 |
| DOC | Document (doctor upload) | 235 |

Demo timeline (newest first, grouped `2026` / `2025` / `2022`):

| Group | Date · type | Title | Org | Doc |
|---|---|---|---|---|
| 2026 | Today · Lab report | Lipid panel — final | Apollo Diagnostics | PDF |
| 2026 | 12 Aug · Consultation | Follow-up — Type 2 diabetes | Dr. R. Menon · City General | — |
| 2026 | 12 Aug · Prescription | Metformin 500 mg renewed | Dr. R. Menon | — |
| 2026 | 03 Jun · Lab report | HbA1c — 6.8% | Apollo Diagnostics | PDF |
| 2026 | 18 Feb · Vaccination | Influenza vaccine | City General | — |
| 2025 | 09–12 Nov · Hospitalization | Appendectomy — discharged | City General Hospital | PDF |
| 2022 | 14 Mar · Diagnosis | Type 2 diabetes (E11.9) | Dr. S. Iyer | — |

The timeline is a consented, read-time composition across tenants (events above span City General, Apollo Diagnostics, and Dr. S. Iyer's org). Records the doctor saves in this consult appear at the top of the current-year group immediately (§9.4). Proposed: tapping an event with an attached document opens the document viewer (each view audited, PRD §13); the prototype does not wire doctor-side event taps.

### 8.4 Reports tab

Toggle row of two pills, `List` / `Compare` (active = ink fill, white text; inactive = white fill, ink text; both states carry a 1px `rgba(34,48,60,0.15)` border). Default: List.

**List view** — report cards: 38×44px PDF thumbnail block (`oklch(0.94 0.02 235)` with `PDF` micro-label), name 14px semibold, `org · date` 12px, trailing lifecycle status pill (colors per §3). Demo data:

| Report | Org | Date | Status |
|---|---|---|---|
| Lipid panel | Apollo Diagnostics | 21 Aug 2026 | Final |
| Thyroid panel (TSH, T4) | Apollo Diagnostics | 21 Aug 2026 | Preliminary |
| HbA1c | Apollo Diagnostics | 03 Jun 2026 | Final |
| Chest X-ray | City General | 10 Nov 2025 | Amended |
| Complete blood count | City General | 09 Nov 2025 | Final |

Lifecycle states surface the report-lifecycle model of PRD §24 in simplified display form: `Final` maps to the FINAL/PUBLISHED stage of the PRD's lifecycle (ORDERED → IN_PROGRESS → COMPLETED → VERIFIED → FINAL → PUBLISHED_TO_PATIENT), `Preliminary` covers the pre-final stages, and `Amended` surfaces the PRD's post-publication correction/versioning requirement. A Preliminary report is visibly not-final at a glance, and an Amended report signals a superseding version exists (see [11-integrations.md](11-integrations.md) and [09-data-model.md](09-data-model.md)).

**Compare view** — side-by-side comparison of two instances of the same panel. Rendered as a card with a 3-column grid (`1.3fr 1fr 1fr`), header row on `oklch(0.96 0.01 235)`: `LIPID PANEL | 14 FEB | TODAY` (11px semibold `rgba(34,48,60,0.55)`; values right-aligned). Rows, 13px, older value at `rgba(34,48,60,0.6)`, newer value semibold with delta arrow:

| Analyte | 14 Feb | Today |
|---|---|---|
| Total cholesterol | 190 | 182 ▼ |
| LDL | 112 | 104 ▼ |
| HDL | 49 | 51 ▲ |
| Triglycerides | 155 | **168 ▲** (amber `oklch(0.55 0.15 60)`, weight 700) |

Comparability footnote beneath the table, 11.5px `rgba(34,48,60,0.5)`: `Both reports FINAL · Apollo Diagnostics · same fasting protocol`. The footnote asserts why the comparison is valid (both final, same source, same protocol). Proposed generalization: Compare pairs the two most recent FINAL instances of the same report type; when no comparable pair exists the Compare pill is disabled with hint "Needs two final reports of the same type". Clinically worsening deltas render in amber bold; improving/neutral deltas in ink.

### 8.5 Meds tab

Card list of prescriptions across the record. Card: name 14px semibold + status pill on the top row; frequency/instructions line 12.5px `rgba(34,48,60,0.6)`; `prescriber · date` 11.5px `rgba(34,48,60,0.45)`. Demo data:

| Medication | Frequency | Status | Prescriber · date |
|---|---|---|---|
| Metformin 500 mg | Twice daily · with meals | Active | Dr. R. Menon · 12 Aug 2026 |
| Atorvastatin 10 mg | Once daily · night | Active | Dr. R. Menon · 12 Aug 2026 |
| Amoxicillin 500 mg | Course completed · post-surgery | Done | Dr. A. Fernandes · Nov 2025 |

Statuses follow PRD §8 (active / completed / discontinued / historical); the prototype shows `Active` (green) and `Done` (neutral).

### 8.6 Bottom action bar

Fixed bar on `#fdfbf7` with top hairline, present on all four tabs: primary dark button `+ Add to record` (flex-grow) → Add to record screen; secondary outlined button `End consult` → §10.

**Acceptance criteria (Patient 360)**

- The allergy banner is visible on all tabs, in normal and emergency sessions, and cannot be scrolled away or dismissed.
- The AI summary card shows the skeleton + `Generating…` state during generation and never shows stale content for a changed slice (cache invalidation per architecture §10.4).
- The brief and every Q&A answer carry the verification disclaimer and derive only from the authorized slice; a question whose answer is not in the record yields an explicit "not in the record" response, never a guess.
- Timeline contents and ordering are identical to the patient's own timeline for the categories the doctor is authorized to see.
- Report status pills render Final/Preliminary/Amended correctly from record state; Compare deltas are computed, not hard-coded, and flag worsening values.
- Every tab view of the 360 is an audited read.

---

## 9. Add to record

Entry: `+ Add to record` in the 360. Header: back link `‹ Priya Sharma` (the patient's name) → 360; title `Add to record` Instrument Serif 26px; type selector — three pills `Diagnosis · Prescription · Report` (active = ink fill/white text). Default type: Diagnosis. (Consultation events are not authored here: the Consultation record for the session is persisted at End consult — §10.1.)

Below the form, in all three modes: primary dark button `Save to Priya's record` (button copy embeds the patient's first name), and beneath it the signing note, centered 11.5px `rgba(34,48,60,0.5)`: `Signed as Dr. R. Menon · added to audit trail`.

Field anatomy: label 12px semibold `rgba(34,48,60,0.55)` above a white input, 12px radius, 1px `rgba(34,48,60,0.18)` border, 14.5px text.

### 9.1 Diagnosis form

| Field | Placeholder | Notes |
|---|---|---|
| Diagnosis | `e.g. Seasonal allergic conjunctivitis` | Free text, required (Proposed) |
| Code (optional) | `ICD-10, e.g. H10.1` | Structured coding hook, PRD §7 |
| Clinical notes | `Findings, plan, advice… or tap Dictate` | 4-row textarea |

The Clinical notes label row carries a right-aligned **`● Dictate`** pill (11.5px semibold, accent text `oklch(0.45 0.13 235)`, 1px `oklch(0.72 0.08 235)` border, 999px radius). Tapping it runs voice dictation and inserts the transcribed text into the notes field, replacing current content, with toast `Dictated note inserted`. Demo transcription: `Patient reports itchy, watery eyes for 5 days. No vision changes. Bilateral conjunctival injection. Plan: antihistamine drops, review in 2 weeks.` Proposed: in product, Dictate toggles a recording state (live waveform/stop affordance) and appends at the cursor; transcription runs on-device or via a PHI-compliant service per [13-non-functional.md](13-non-functional.md).

### 9.2 Prescription form and allergy conflict warning

| Field | Placeholder |
|---|---|
| Medication | `e.g. Cetirizine 10 mg` |
| Dosage & frequency | `e.g. once daily · 14 days` |
| Instructions | `With food, avoid alcohol…` (3-row textarea) |

**Allergy conflict warning.** As the doctor types the medication name, the input is checked against the patient's recorded allergies. In the demo, any input matching the penicillin class — case-insensitive substring `penicillin`, `amoxicillin`, or `augmentin` — immediately renders a warning card between the Medication field and the rest of the form: `oklch(0.95 0.03 25)` background, 1.5px `oklch(0.7 0.14 25)` border, 12px radius, 12.5px text in `oklch(0.4 0.15 25)`:

> **Allergy conflict:** Priya has a recorded severe penicillin allergy (2019). Saving will require an override reason and is flagged to compliance.

The warning appears live (per keystroke) and disappears if the input no longer matches. Product behavior: matching is a drug-class check against the patient's allergy list (the demo's three names stand in for a proper class/ingredient mapping — Proposed: RxNorm/ATC-backed class matching via the terminology service). Saving while the warning is active requires an **override**: the save is blocked until the doctor supplies an override reason (Proposed UI: a required "Override reason" field or confirmation sheet appears on Save — the prototype states the requirement in the warning copy but does not render the field), and the resulting record plus audit event are flagged to compliance.

### 9.3 Report form

- **Upload dropzone:** dashed 1.5px `rgba(34,48,60,0.25)` border, 14px radius, centered content — PDF thumbnail glyph, `Upload report or photo` 14px semibold, and the constraint line 12px `rgba(34,48,60,0.5)`: `PDF, JPG, DICOM · scanned for malware`. Tapping opens the platform file/camera picker (Proposed: camera, photo library, and files).
- **Title field**, placeholder `e.g. Clinical photo — left eye`.

Accepted formats: PDF, JPG, DICOM (prototype copy; architecture §9 Documents — "PDF, images, DICOM"; PRD §6 requires support for common document/image formats). Every upload passes the malware-scanning pipeline before the document becomes visible on the record (PRD §17, architecture §9 Documents: upload via pre-signed URL, `scan_status: clean|blocked`). Proposed states the prototype omits: an "Uploading… / Scanning…" progress state after save, and a blocked-file error ("File failed security scanning and was not added") with the event audited.

### 9.4 Save behavior

Tapping `Save to Priya's record`:

1. Persists the record via the clinical API, attributed to the authoring doctor and organization and signed (`Signed as Dr. R. Menon`); tenant-owned by the doctor's organization (see [09-data-model.md](09-data-model.md)).
2. Prepends a new event to the patient's timeline in the current-year group (`2026`), dated `Today`:

```text
Diagnosis     → type "Diagnosis",    chip DX (hue 25),  title = "<diagnosis> (<code>)" (code suffix only if entered),
                org "Dr. R. Menon · City General"
Prescription  → type "Prescription", chip RX (hue 150), title = "<medication> — <dosage & frequency>",
                org "Dr. R. Menon"
Report        → type "Document",     chip DOC (hue 235), title = <title>, PDF badge attached,
                org "Dr. R. Menon · City General"
```

   (Prototype fallback titles when fields are left empty: `Seasonal allergic conjunctivitis`, `Cetirizine 10 mg — once daily · 14 days`, `Report uploaded`. Product: required fields validate instead — Proposed.)
3. Navigates back to Patient 360 on the **Timeline tab**, where the new event is visible at the top of the 2026 group; the form is cleared.
4. Toast: `Saved to record · signed & audit logged` — or, when the save carried an allergy override, `Saved with allergy override — flagged to compliance`.
5. Downstream: the event also appears in the patient's own timeline, and the patient is notified of the new record per PRD §16 (see [02-patient-app.md](02-patient-app.md)).

**Acceptance criteria (Add to record)**

- Switching type pills preserves the screen but swaps the form; unsaved input is kept per form during the visit to this screen (prototype keeps a single field-set; Proposed: per-form retention) and cleared after save or after leaving the consult.
- The allergy warning appears within one keystroke of a matching medication name and clears when the name no longer matches.
- A prescription matching a recorded allergy class cannot be saved without an override reason; the saved record and audit entry carry the compliance flag.
- An uploaded document is not visible to anyone until malware scanning marks it clean.
- Every save writes an audit event with author, role, patient, org, timestamp, and resource (PRD §13).

---

## 10. End consult and the consult session model

### 10.1 End consult

Tapping `End consult` in the 360 action bar:

- Closes the consult session and returns to Home.
- Resets consult-scoped UI state: emergency banner cleared, open ask-the-record answer cleared, active tab resets to Overview for the next session.
- Toast: `Consult ended · 1 session recorded`.
- Writes a session-end audit event (Proposed: with session duration and counts of reads/writes performed).
- **Proposed — persists the Consultation record** (PRD §3.2 "Record consultations and diagnoses"; PRD §5 consultation timeline events, e.g. "Follow-up — Type 2 diabetes"): ending the consult creates a Consultation timeline event ([09-data-model.md](09-data-model.md) §6.2) attributed to the doctor and organization, carrying a reason line (entered at end-of-consult or derived from the primary diagnosis), the consult notes, and `related_diagnosis_ids` / `related_prescription_ids` linking every record saved during the session. API: `POST /v1/patients/{id}/encounters` ([12-api-and-interoperability.md](12-api-and-interoperability.md) §3.3). The event appears in both the doctor and patient timelines with the `CON` chip. A consult with no saved records and no notes writes no Consultation event (session audit still records the access).

Ending the consult ends the *session*, not the *grant*: the consult grant persists to its expiry (e.g. Priya's runs to 12 Sep 2026), which is why she remains openable from Recent patients afterwards. An emergency session's constrained grant, by contrast, should not outlive the session (Proposed, consistent with "limited to the emergency profile" — post-hoc review governs any follow-up access).

### 10.2 Consult session model

A **consult session** is the unit of doctor access to one patient's record. It is:

| Property | Specification |
|---|---|
| Scoped | Bound to one doctor, one patient, one tenant context, and the authorized slice determined by the policy engine (RBAC × ABAC × consent categories). QR token resolution is session-bound (architecture §8) — a resolved token cannot be replayed into another session. |
| Timed | The session has a bounded duration, displayed as a live countdown in the 360 header pill (`Session 28:44 · audited`). Proposed: 30-minute window (consistent with the 28:44 display shortly after scan), extendable by activity or re-scan; on expiry the 360 locks and returns to Home with a "Session expired — re-scan to continue" notice. Session lifetime is independent of the consult grant duration (default 30 days, LLD `consultGrant(30d, categories)`). |
| Audited | Session start (scan/grant/view events), every tab view, every document open, every AI generation (prompt, retrieved slice, output — architecture §10.5), every write, break-glass entry, and session end are appended to the tamper-resistant audit log (PRD §13). The header pill's `audited` label and the login/save footnotes make this visible to the doctor at all times. |

Session state (timer, bound token) lives in the server-side session store (Redis-class, TTL-bound, no durable PHI — architecture §13); the client renders but does not own the countdown.

**Acceptance criteria**

- End consult always produces exactly one session-recorded confirmation and leaves no consult UI state behind.
- A second scan of the same patient starts a new session and new audit trail segment; it does not resume the old one.
- Session expiry closes access even if the app stays open; continued use requires re-scan.
- Every session is reconstructable from the audit log: who, whom, when, what was viewed, what was written, and under which grant (normal or emergency).

---

## 11. Cross-references

| Topic | Spec |
|---|---|
| Design tokens, type scale, components | [01-design-system.md](01-design-system.md) |
| Patient-side mirror of these flows (QR display, consent approval, access log, notifications) | [02-patient-app.md](02-patient-app.md) |
| AI brief, ask-the-record pipeline, disclaimers, caching, safety | [05-ai-features.md](05-ai-features.md) |
| Doctor SSO/MFA, session tokens, professional verification | [06-identity-auth.md](06-identity-auth.md) |
| Consent grants, categories, durations, break-glass policy | [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| QR token issuance, rotation, resolution, anti-replay | [08-qr-subsystem.md](08-qr-subsystem.md) |
| Entities: ConsentGrant, Report lifecycle, Document scan status | [09-data-model.md](09-data-model.md) |
| Report Final/Preliminary/Amended provenance from integrations | [11-integrations.md](11-integrations.md) |
| `/qr`, `/patients/{id}/summary`, clinical write endpoints | [12-api-and-interoperability.md](12-api-and-interoperability.md) |
| SLOs (QR p95 < 500 ms, 360 load < 1 s, brief p95 < 2.5 s), device security | [13-non-functional.md](13-non-functional.md) |
