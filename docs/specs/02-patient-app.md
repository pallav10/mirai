# Patient app

Purpose: implementation-ready specification of every patient-facing screen, state, interaction, and edge case of the Atlas mobile app (iOS and Android).

Sources: `design/Atlas App.dc.html` (patient screens and prototype logic — authoritative for layout, copy, colors, interactions), `design/uploads/Atlas_Project_Requirements.md` §3.1, 4, 5, 6, 8, 9, 10, 12, 13, 14, 16, 19, 24, 27, 33, `design/Atlas Directions.dc.html` (direction "1a Meridian"), `design/ios-frame.jsx` / `design/android-frame.jsx` (device frames).

Related specs: visual tokens in [01-design-system.md](01-design-system.md); AI summary content rules in [05-ai-features.md](05-ai-features.md); OTP/biometric/passkey mechanics in [06-identity-auth.md](06-identity-auth.md); consent semantics in [07-consent-and-access-control.md](07-consent-and-access-control.md); QR token behavior in [08-qr-subsystem.md](08-qr-subsystem.md); entities in [09-data-model.md](09-data-model.md); report sync/provenance in [11-integrations.md](11-integrations.md).

---

## 1. Scope and platform

The patient app is the patient's window onto their own Atlas record (PRD §3.1): view medical information, reports, diagnoses, prescriptions and discharge summaries; present the Atlas QR; see and control who has access; manage profile basics. Patients never author clinical data in this app — clinical records arrive from doctors and integrations.

- Platforms: native-feel mobile app for iOS and Android. The prototype renders identical screen content inside an iOS 26 liquid-glass frame and a Material 3 Android frame; screen structure, copy, and behavior are the same on both platforms. Platform-specific chrome (status bar, home indicator, back gestures) follows the platform.
- Reference viewport: 390 × 844 px. All layouts use a 22 px horizontal screen gutter (28 px on the onboarding screens) and must adapt fluidly to other device widths.
- Visual language: direction "1a Meridian" — warm paper background `#faf7f1`, ink `#22303c`, ocean-blue accent `oklch(0.52 0.13 235)` (link/active shade `oklch(0.45 0.13 235)`), Instrument Serif for display headings, Instrument Sans for body. Full token table in [01-design-system.md](01-design-system.md).
- Sample data used throughout this spec is the canonical demo persona: Priya Sharma, 34 F, Atlas ID `ATL-82X92K`, B+, severe penicillin allergy, Type 2 diabetes (E11.9), Metformin 500 mg BD + Atorvastatin 10 mg OD, appendectomy Nov 2025, ABHA `priya@abdm`, family members Aarav and Kamala Sharma.

### Acceptance criteria
- The app ships on both iOS and Android with feature-identical patient flows.
- All screens render correctly at 390 pt width and reflow at other common widths without horizontal scrolling.

---

## 2. Navigation map

```text
welcome ──"Continue with OTP"──▶ otp ──"Verify & continue"──▶ home
   └──────"Sign in with Face ID" (biometric) ─────────────────▶ home

Tab bar (visible on home, timeline, records, profile):
  Home · Timeline · [QR center button] · Records · Profile

home ──▶ qr (full-screen, no tab bar; "‹ Close" returns home)
home ──▶ notifs ("‹ Back" returns home)
home ──"All" (medication)──▶ records (Prescriptions tab)
home ──"Timeline"──▶ timeline
home ──recent item tap──▶ report
home ──access banner──▶ access
timeline ──event with document tap──▶ report
records (Reports tab) ──row tap──▶ report ("‹ Records" returns)
records (Discharge tab) ──row tap──▶ discharge ("‹ Records" returns)
profile ──▶ access ──▶ consent ("‹ Back" chain returns)
profile ──▶ notifs · emergency (security · language rows are inert
           in the prototype — Proposed targets, see §12)
notifs ──"Review request"──▶ consent
```

| Screen key | Title / label | Entry points | Tab bar |
|---|---|---|---|
| `welcome` | Patient welcome | App launch (signed out) | No |
| `otp` | Enter the code | Welcome → Continue with OTP | No |
| `home` | Patient home | OTP verify, biometric sign-in, tab | Yes |
| `qr` | QR present (full-screen dark) | Tab-bar center button, home QR card | No |
| `timeline` | Timeline | Tab, home "Timeline" link | Yes |
| `records` | Records | Tab, home "All" link (→ Prescriptions tab) | Yes |
| `report` | Report detail (Lipid panel) | Timeline, Records, home recent item | No (push) |
| `discharge` | Hospitalization detail | Records → Discharge tab | No (push) |
| `profile` | Patient profile | Tab | Yes |
| `access` | Access & consent | Profile row, home access banner | No (push) |
| `consent` | Consent request | Access pending card, notification CTA | No (push) |
| `notifs` | Notifications | Home bell, profile row | No (push) |
| `emergency` | Emergency card | Profile row; also lock screen (see §16) | No (push) |

Push screens use a text back affordance at the top-left of the content area (`‹ Back`, `‹ Records`, `‹ Close`), 14 px, `rgba(34,48,60,0.6)` (70% white on the dark QR screen). Proposed: platform back gesture / hardware back performs the same navigation.

### Tab bar

Background `#fdfbf7` with a 1 px top border `rgba(34,48,60,0.1)`; padding 12 px top, 30 px bottom (home-indicator safe area). Four labeled items — Home, Timeline, Records, Profile — each a 5 px dot indicator above a 10.5 px semibold label. Active item: ocean `oklch(0.45 0.13 235)`; inactive: `rgba(34,48,60,0.5)`. Center element: a 46 px ink `#22303c` circle raised 18 px above the bar (shadow `0 4px 10px rgba(34,48,60,0.3)`) containing a white QR glyph (18 px rounded square outline); tapping it opens the full-screen QR present screen. The tab bar is hidden on welcome, otp, qr, and all push screens.

### Acceptance criteria
- Every edge in the map above is reachable by tap and returns as described.
- The tab bar appears on exactly Home, Timeline, Records, Profile; the QR button opens the QR screen from any tab.

---

## 3. Onboarding — Welcome screen

Centered hero block on paper background, tab bar hidden:

- Compass-ring Atlas logo, 72 px.
- Display heading, Instrument Serif 38 px, line-height 1.1: `Your health,` / `in one place`.
- Subtitle, 14.5 px, `rgba(34,48,60,0.6)`, max-width 260 px, centered: `One identity, one QR, your complete medical history — shared only with doctors you allow.`
- Language chips row (11.5 px pills): `English` (selected: ink background, white text, semibold), `हिंदी`, `मराठी` (unselected: white background, `rgba(34,48,60,0.15)` border). Tapping a chip switches the app language immediately (multi-language is an approved requirement; string catalogs per [01-design-system.md](01-design-system.md)). Proposed: the choice persists to the device and pre-selects on next launch; it can be changed later from Profile → Language.

Bottom-anchored form column (10 px gap):

- Mobile number input: white field, 12 px radius, `rgba(34,48,60,0.18)` border, 15 px text, placeholder `Mobile number`. Demo value `+91 98220 41175`. Proposed: input restricted to phone characters with country-code prefix; default `+91` (India-first).
- Primary button `Continue with OTP` (ink background, white 15 px semibold, 12 px radius, 15 px vertical padding) → sends an OTP to the entered number and pushes the OTP screen. OTP issuance rules in [06-identity-auth.md](06-identity-auth.md).
- Text button `Sign in with Face ID` (ocean, 13.5 px semibold) → biometric authentication for a returning enrolled user; success lands directly on Home. Proposed: label adapts per platform/capability — `Sign in with Face ID` (iOS Face ID), `Sign in with fingerprint` / biometric prompt on Android; hidden entirely if no biometric credential is enrolled. Passkey sign-in also lives behind this path (see [06-identity-auth.md](06-identity-auth.md)).
- Footnote, 12 px, `rgba(34,48,60,0.5)`, centered: `Works with ABHA (ABDM) · encrypted · you control access`.

Edge cases — Proposed (not shown in prototype): invalid/short number disables the OTP button; network failure on OTP request shows an inline error under the field and leaves the user on Welcome; biometric failure/cancel falls back to the OTP path.

### Acceptance criteria
- A new user can request an OTP from this screen; a returning enrolled user can reach Home via biometrics without an OTP.
- All three languages are selectable and the selection visibly changes the active chip.
- The exact hero copy, subtitle, and ABHA footnote strings above are used in the English locale.

---

## 4. OTP verification

Push screen from Welcome (`‹ Back` returns without consuming the code).

- Heading: Instrument Serif 28 px, `Enter the code`; below it 13.5 px `rgba(34,48,60,0.6)`: `Sent to +91 98220 41175` (the number actually entered).
- Four OTP boxes, 56 × 64 px, white, 12 px radius, 24 px semibold digits. Filled boxes carry a 1.5 px ocean border; the active (focused) box shows a muted `•` caret; untouched boxes have a 1 px `rgba(34,48,60,0.2)` border. Prototype depicts a 4-digit code with `4 7` entered.
- Primary button `Verify & continue` (ink, full-width) → verifies the code; success lands on Home.
- Resend line, 13 px, `rgba(34,48,60,0.5)`, centered: `Resend code in 0:24` — a live countdown; when it reaches 0:00 it becomes a tappable `Resend code` action (Proposed; the prototype shows only the counting state). Resend limits and OTP expiry in [06-identity-auth.md](06-identity-auth.md).

Edge cases — Proposed: wrong code shakes the boxes and shows an inline error, clearing input; too many failures locks the attempt per auth policy; auto-advance focus per digit; auto-submit on fourth digit is acceptable but the explicit button must remain.

### Acceptance criteria
- The screen echoes the exact phone number the OTP was sent to.
- A correct code navigates to Home; Back returns to Welcome with the entered number preserved.
- The resend countdown is visible and accurate.

---

## 5. Home

Scrollable screen; tab bar visible. Top-to-bottom composition:

**Header.** Left: date line 13 px `rgba(34,48,60,0.55)` (`Wednesday, 21 August`), then greeting in Instrument Serif 30 px over two lines: `Good morning,` / `{first name of active profile}` (demo: `Priya`). Proposed: greeting varies by local time (Good morning/afternoon/evening). Right: notification bell — 40 px white circle, `rgba(34,48,60,0.12)` border, ring glyph; an unread badge (10 px red dot `oklch(0.55 0.18 25)` with 2 px paper border) appears at the top-right when any unread notification exists. Tap → Notifications.

**Family profile chips.** Horizontally scrollable row of pill chips, one per family member on this login (demo: `Priya`, `Aarav`, `Kamala`). Each chip: 26 px initials avatar + first name, 12 px semibold. Selected chip: ink background, white text, avatar on `rgba(255,255,255,0.2)`; unselected: white background, ink text, avatar on `oklch(0.9 0.03 235)` with ocean initials. Tapping a chip switches the active profile — see §6.

**QR card.** White card (16 px radius, 1 px `rgba(34,48,60,0.1)` border, subtle shadow) with an 84 px QR thumbnail in a bordered inset, title `My Atlas QR` (15 px semibold), the Atlas ID `ATL-82X92K` in 12 px monospace `rgba(34,48,60,0.55)`, and an underlined ocean link `Present to doctor`. The entire card taps through to the full-screen QR screen.

**AI Summary card.** Gradient card `linear-gradient(135deg, oklch(0.96 0.02 235), oklch(0.94 0.03 260))`, border `oklch(0.86 0.05 245)`, 16 px radius. Header row: diamond spark icon (`oklch(0.5 0.14 260)`), label `AI SUMMARY` (11 px semibold, letter-spacing 0.06em, `oklch(0.42 0.12 260)`), right-aligned freshness stamp 10.5 px `Updated today, 9:15`. Body, 13.5 px / 1.55: demo copy `Your new lipid panel looks mostly good — cholesterol is in range, but triglycerides are slightly high. Your diabetes remains well controlled (HbA1c 6.8%). Keep taking both medications; Dr. Menon may discuss diet at your 12 Sep follow-up.` Footer disclaimer, 10.5 px `rgba(34,48,60,0.45)`: `Generated from {n} records · not medical advice — ask your doctor` (demo n = 7). Generation triggers, record-count rule, regeneration cadence, and the plain-language style contract are specified in [05-ai-features.md](05-ai-features.md). The disclaimer line is mandatory and may not be truncated. States follow [05-ai-features.md](05-ai-features.md) §2.5: while a new summary generates, the previous summary stays visible with its old stamp (no skeleton on Home); for a patient with no published records the card is hidden entirely — there is nothing to summarize.

**Today's medication.** Section heading Instrument Serif 19 px `Today's medication` with a right-aligned 12.5 px link `All` → Records screen, Prescriptions tab. Rows (divider `rgba(34,48,60,0.08)`): dose dot + `{Medication strength}` 14 px semibold + ` · {instruction}` 12.5 px muted + right-aligned time 12 px. Demo rows: `Metformin 500 mg · with breakfast — 8:00` with a filled ocean dot, `Atorvastatin 10 mg · after dinner — 21:00` with a hollow 1.5 px-bordered dot. Proposed semantics: filled dot = dose already taken/passed today, hollow = upcoming; derived from active prescriptions (PRD §8) and their frequency/instructions. The list shows only medications with status Active.

**Recent.** Heading Instrument Serif 19 px `Recent` with right link `Timeline` → Timeline. A short activity list (2 items) along a 2 px `oklch(0.85 0.05 235)` left rule: each item shows an 11 px context line (`Today · Apollo Diagnostics`), a 14 px semibold title (`Lipid panel report available`), and — when the item has an open target — an ocean action line (`View report`) tapping through to the report detail. Second demo item: `12 Aug · Dr. R. Menon` / `Consultation — follow-up, Type 2 diabetes` (no tap target in prototype). Recent shows the newest timeline events; Proposed: the 2 most recent.

**Access banner.** Full-width tinted banner `oklch(0.94 0.02 235)`, 14 px radius: ocean status dot, text 13 px `**2 providers** currently have access to your records` (provider count bold, computed from active grants), chevron `›`. Tap → Access & consent.

### Acceptance criteria
- Greeting shows the active family member's first name and updates on profile switch.
- The unread badge on the bell reflects actual unread notifications.
- AI Summary card always carries the record-count + "not medical advice" disclaimer line.
- Medication rows are derived from active prescriptions; the "All" link lands on the Prescriptions tab specifically.
- The access banner count equals the number of currently active access grants and opens Access & consent.

---

## 6. Family profiles

One login (one mobile number/biometric identity) can hold multiple patient profiles — an approved requirement (family profiles under one login). Demo family: Priya (account owner), Aarav Sharma, Kamala Sharma.

Model (see [09-data-model.md](09-data-model.md) and [06-identity-auth.md](06-identity-auth.md)):

- Each family member is a full, distinct global patient identity with their own Atlas ID, QR, records, consents, and audit trail. The login account holds links to the profiles it may act for.
- Switching the active profile re-scopes the entire app — Home greeting and content, QR screen, Timeline, Records, Profile, Access & consent, Notifications — to that member. The chips row remains visible on Home for switching back.
- In the prototype only Priya's data is populated; selecting Aarav or Kamala shows the toast `Viewing Aarav Sharma — demo data shows Priya` (respectively Kamala). In the product this toast does not exist — the switch actually loads the member's data. Proposed toast on switch instead: `Now viewing {full name}`.
- Proposed (not depicted): adding/removing family members, and guardianship rules for minors/elders, are managed outside this screen set and specified in [06-identity-auth.md](06-identity-auth.md); the chips row simply reflects the linked set.

### Acceptance criteria
- Selecting a family chip re-scopes every screen, including the QR code presented, to that member.
- Each member's QR resolves to that member's own Atlas identity, never the account owner's.

---

## 7. QR present

Full-screen modal, dark ink background `#22303c`, white text, no tab bar. `‹ Close` (top-left, `rgba(255,255,255,0.7)`) returns to Home.

Centered column:

- Patient name, Instrument Serif 26 px: `Priya Sharma` (active profile's full name).
- QR code: 250 × 250 px white rounded card (20 px radius, 18 px padding, shadow `0 10px 40px rgba(0,0,0,0.35)`) containing the QR at maximum brightness for scanability.
- Atlas ID, 14 px monospace, `rgba(255,255,255,0.8)`: `ATL-82X92K`.
- Token status pill: translucent `rgba(255,255,255,0.1)` pill, 12.5 px, with a green dot `oklch(0.8 0.15 150)`: `Secure token · rotates in 4:52` — a live countdown to the next token rotation. When it elapses, the QR re-renders with a fresh token and the counter resets (rotation period, expiry, revocation and anti-replay in [08-qr-subsystem.md](08-qr-subsystem.md)). The QR encodes only a secure opaque token resolvable by Atlas — never medical data (PRD §10, §33).
- Footer, 12.5 px `rgba(255,255,255,0.55)`, max-width 280 px, centered: `This code identifies you — it contains no medical data. The doctor sees only what you've authorized.`

Proposed behaviors: screen brightness is raised to maximum while this screen is open and restored on close; screenshot inhibition is unnecessary (tokens expire server-side), but the token countdown must reflect real server-issued expiry; if the device is offline, the last issued token displays with its remaining validity, and after expiry the screen shows a "reconnect to refresh your code" state.

### Acceptance criteria
- The rendered QR contains only the rotating access token; decoding it yields no PHI.
- The countdown matches the token's actual remaining validity and the code refreshes seamlessly at rotation.
- Name shown always matches the active family profile.

---

## 8. Timeline

Tab screen. Header: Instrument Serif 28 px `Timeline`; subtitle 12.5 px `rgba(34,48,60,0.5)`; filter chip row below (wraps if needed).

**Filter chips** — `All`, `Consultations`, `Reports`, `Hospital` — pill style: selected = ink background/white text, unselected = white/ink with `rgba(34,48,60,0.15)` border. Filter mapping by event type:

| Chip | Event types included |
|---|---|
| All | every event |
| Consultations | Consultation, Diagnosis |
| Reports | Lab report, Document |
| Hospital | Hospitalization |

(Vaccination and Prescription events appear only under All in the demo data set; the mapping table is the product rule.)

**Subtitle rules** (exact formats):

- All filter: `{count} events · {firstYear} – today · from {orgCount} organizations` — demo: `7 events · 2022 – today · from 3 organizations`. `{orgCount}` is the number of distinct source organizations contributing events.
- Any other filter: `{count} event · {label}` when count is 1, `{count} events · {label}` otherwise, where label is `consultations`, `reports`, or `hospital`. Demo: Hospital → `1 event · hospital`; Consultations → `2 events · consultations`.

**Year groups.** Events are grouped by year, newest first (demo groups: 2026, 2025, 2022). Group header: 11 px semibold letter-spaced year label `rgba(34,48,60,0.45)` with a 1 px rule filling the rest of the row.

**Event cards.** White card, 14 px radius, 1 px border, laid out as: 38 px rounded type chip + text column + optional `PDF` pill. The type chip is a 9.5 px bold abbreviation on a hue-tinted square — background `oklch(0.93 0.045 {h})`, text `oklch(0.4 0.12 {h})`:

| Chip | Event type | Hue h |
|---|---|---|
| `LAB` | Lab report | 235 |
| `CON` | Consultation | 280 |
| `RX` | Prescription | 150 |
| `VAX` | Vaccination | 190 |
| `HSP` | Hospitalization | 60 |
| `DX` | Diagnosis | 25 |
| `DOC` | Document | 235 |

Text column: 11 px context line `{date} · {type}` (`Today · Lab report`), 14 px semibold title, 12 px muted organization line. Events with an attached document show a `PDF` pill (10 px semibold, `oklch(0.94 0.02 235)` background, `oklch(0.42 0.12 235)` text). Demo event set:

| Date | Type | Title | Organization | Doc | Year |
|---|---|---|---|---|---|
| Today | Lab report | Lipid panel — final | Apollo Diagnostics | PDF | 2026 |
| 12 Aug | Consultation | Follow-up — Type 2 diabetes | Dr. R. Menon · City General | — | 2026 |
| 12 Aug | Prescription | Metformin 500 mg renewed | Dr. R. Menon | — | 2026 |
| 03 Jun | Lab report | HbA1c — 6.8% | Apollo Diagnostics | PDF | 2026 |
| 18 Feb | Vaccination | Influenza vaccine | City General | — | 2026 |
| 09–12 Nov | Hospitalization | Appendectomy — discharged | City General Hospital | PDF | 2025 |
| 14 Mar | Diagnosis | Type 2 diabetes (E11.9) | Dr. S. Iyer | — | 2022 |

**Tap-through.** In the prototype only the lipid panel event opens the report detail. Product rule: any event with an attached document opens its detail screen (lab report → report detail; hospitalization → hospitalization detail); events without a detail target are inert. Proposed: consultation/diagnosis detail screens are a later addition; until then those cards are non-interactive.

**Empty state — Proposed:** a filter with no matches keeps the subtitle (`0 events · {label}`) and shows a quiet centered message: "Nothing here yet." New records arriving via integration appear in the timeline as soon as they are published to the patient (PRD §19 flow).

### Acceptance criteria
- Filter chips show exactly the mapped event types and the subtitle follows the count/pluralization/label format above, including the singular `1 event · hospital` case.
- Events are grouped by year, newest first, with the correct chip abbreviation and hue per type.
- Tapping a documented lab event opens the report detail; tapping the hospitalization opens the hospitalization detail.

---

## 9. Records

Tab screen. Header: Instrument Serif 28 px `Records`; below it a segmented chip row: `Reports` · `Prescriptions` · `Discharge` — 12.5 px semibold pills, 6 × 14 px padding (selected = ink background/white text, unselected = white with `rgba(34,48,60,0.15)` border); one always selected; default Reports. Entering via Home's medication "All" link preselects Prescriptions.

### 9.1 Reports tab

List of report cards: 38 × 44 px `PDF` document glyph (tinted `oklch(0.94 0.02 235)` / ocean text), name 14 px semibold, `{org} · {date}` 12 px muted, right-aligned lifecycle status pill (10.5 px semibold). Demo list:

| Report | Organization | Date | Status |
|---|---|---|---|
| Lipid panel | Apollo Diagnostics | 21 Aug 2026 | Final |
| Thyroid panel (TSH, T4) | Apollo Diagnostics | 21 Aug 2026 | Preliminary |
| HbA1c | Apollo Diagnostics | 03 Jun 2026 | Final |
| Chest X-ray | City General | 10 Nov 2025 | Amended |
| Complete blood count | City General | 09 Nov 2025 | Final |

Status pill palette (shared app-wide):

| Status | Background | Text |
|---|---|---|
| Final / Active | `oklch(0.93 0.05 150)` | `oklch(0.42 0.12 150)` |
| Preliminary | `oklch(0.93 0.06 80)` | `oklch(0.45 0.12 60)` |
| Amended | `oklch(0.94 0.02 235)` | `oklch(0.42 0.12 235)` |
| Revoked | `oklch(0.94 0.04 25)` | `oklch(0.45 0.15 25)` |
| Done / other | `rgba(34,48,60,0.08)` | `rgba(34,48,60,0.6)` |

Statuses surface the report lifecycle of PRD §24: patients only ever see reports that have reached publication; `Preliminary` marks a published-but-not-final result, `Amended` marks a corrected report where an updated version supersedes an earlier one (provenance retained). Tapping a row opens that report's detail screen (the prototype wires all rows to the one built detail, §10; the product opens the corresponding report).

### 9.2 Prescriptions tab

Prescription cards (no document glyph): name 14 px semibold + status pill on the first row, frequency line 12.5 px muted, `{prescriber} · {date}` 11.5 px faint. Demo list:

| Prescription | Frequency line | Status | Prescriber · date |
|---|---|---|---|
| Metformin 500 mg | Twice daily · with meals | Active | Dr. R. Menon · 12 Aug 2026 |
| Atorvastatin 10 mg | Once daily · night | Active | Dr. R. Menon · 12 Aug 2026 |
| Amoxicillin 500 mg | Course completed · post-surgery | Done | Dr. A. Fernandes · Nov 2025 |

Statuses map to PRD §8's active/completed/discontinued/historical distinction (`Active`, `Done` shown; Proposed: `Discontinued` uses the Revoked palette).

### 9.3 Discharge tab

Document-style cards, one per discharge summary. Demo: `Discharge summary — appendectomy` / `City General Hospital · 12 Nov 2025`, chevron `›`. Tap → Hospitalization detail (§11).

**Empty states — Proposed:** each tab with no items shows "No {reports/prescriptions/discharge summaries} yet" with a one-line explanation that records added by your doctors and hospitals appear here automatically.

### Acceptance criteria
- The three tabs switch content in place without losing scroll position of the screen header.
- Every report row shows its lifecycle status with the exact pill palette above.
- An amended report is visibly distinguished from a final one in the list.

---

## 10. Report detail (reference: Lipid panel)

Push screen from Records/Timeline/Home; `‹ Records` returns. This screen is the template for all lab report details; the lipid panel instance fixes the layout.

- Title: Instrument Serif 26 px `Lipid panel`; meta line 13 px: `Apollo Diagnostics · 21 Aug 2026 · Final` with the status word colored per status (Final → `oklch(0.5 0.12 150)`, semibold).
- **Results card**: white card, 16 px radius, one row per analyte (13.5 px label left, 13.5 px semibold value right, hairline dividers):

| Analyte | Value | Flag |
|---|---|---|
| Total cholesterol | 182 mg/dL | — |
| LDL | 104 mg/dL | — |
| HDL | 51 mg/dL | — |
| Triglycerides | 168 mg/dL ▲ | High — value rendered in amber `oklch(0.55 0.15 60)` with `▲` marker |

Out-of-range values carry the amber color and a directional marker; in-range values render in ink. Reference ranges come from the source report ([11-integrations.md](11-integrations.md)).

- **Provenance block**: tinted panel `oklch(0.94 0.02 235)`, 12 px text, exactly this structure (PRD §27):

```text
Provenance
Source: Apollo LIS · LAB-928372
Verified 21 Aug 2026, 9:02 · Imported 9:03
Status: FINAL · published to patient
```

- **Trend card** (shown when Atlas holds a series for a tracked analyte relevant to this patient — demo: HbA1c): white card headed `TREND — HBA1C` (11 px semibold letter-spaced) with a right-aligned trend verdict `Improving` in green. Column chart of the last three values, each column labeled with value above and period below, bars deepening in ocean saturation toward the newest value: `7.4` (Dec 24, `oklch(0.85 0.05 235)`), `7.0` (Mar 25, `oklch(0.72 0.09 235)`), `6.8` (Jun 26, `oklch(0.52 0.13 235)`). Chart is display-only.
- **Action row**: two equal buttons — `Download PDF` (white, 1 px ink-tinted border) and `Share` (ink, white text). Proposed behavior: Download saves/opens the source PDF of the report; Share opens the platform share sheet with the PDF. Both actions are recorded in the audit trail as document download/share events (PRD §13) and surface in the patient's access log.

Amended reports — Proposed: an amended report's detail shows the `Amended` status in the meta line and a notice linking to the superseded version, per PRD §24's version/provenance requirement.

### Acceptance criteria
- Out-of-range analytes are visually flagged (amber + marker); in-range analytes are not.
- The provenance block shows source system, source record ID, verified and imported timestamps, and status for every synchronized report.
- The trend card renders only when a series exists, with values and periods matching stored results (7.4 → 7.0 → 6.8 for the demo).
- Download and Share produce audit log entries.

---

## 11. Hospitalization / discharge detail

Push screen from Records → Discharge; `‹ Records` returns. Covers PRD §9.

- Title: Instrument Serif 26 px `Hospitalization`; meta line: `City General Hospital · 09 – 12 Nov 2025` (admission – discharge dates).
- **Key facts card** (label/value rows, hairline dividers):

| Field | Demo value |
|---|---|
| Reason for admission | Acute appendicitis |
| Procedure | Laparoscopic appendectomy |
| Doctors | Dr. A. Fernandes (Surgery) |
| Discharge condition | Stable, recovered — rendered green `oklch(0.5 0.12 150)` |
| Meds at discharge | Amoxicillin 500 mg · 5 days |

- **Follow-up instructions card**: white card headed `FOLLOW-UP INSTRUCTIONS` (11 px semibold letter-spaced), body 13.5 px / 1.6. Demo copy: `Wound review in 10 days. Avoid heavy lifting for 4 weeks. Resume Metformin from day 2. Report fever or wound discharge immediately.`
- **Action row**: `Download summary` (outline) and `Share` (ink) — same audit-logged semantics as the report detail actions.

### Acceptance criteria
- All PRD §9 fields present in the source discharge summary render in the facts card; absent fields collapse rather than showing blanks (Proposed).
- Discharge condition is color-coded when positive; follow-up instructions render verbatim from the record.

---

## 12. Profile

Tab screen. Header: 56 px initials avatar (`PS`, `oklch(0.9 0.03 235)` background, ocean initials), name in Instrument Serif 24 px, Atlas ID in 12.5 px monospace muted.

**Demographics card** (label/value rows):

| Field | Demo value |
|---|---|
| Date of birth | 14 May 1992 (34) |
| Blood group | B+ |
| Allergies | Penicillin — rendered red `oklch(0.5 0.16 25)` |
| Conditions | Type 2 diabetes |
| Emergency contact | Aman S. · +91 98111 20844 |
| ABHA address | `priya@abdm` (monospace) + green `Linked` pill (`oklch(0.93 0.05 150)` / `oklch(0.42 0.12 150)`) |

The ABHA row reflects ABDM linkage state; an unlinked account shows a link affordance instead (Proposed — see [06-identity-auth.md](06-identity-auth.md) for the linking flow). Editing demographics is a PRD §3.1 capability ("manage basic profile information"); Proposed: rows other than Atlas ID/ABHA open an edit sheet; clinical fields (allergies, conditions) are read-only here because they originate from clinical records.

**Settings links** (white cards, title + right hint + `›`):

| Row | Right hint | Target |
|---|---|---|
| Access & consent | `2 active ›` (live count of active grants) | Access & consent (§13) |
| Notifications | `1 new ›` (live unread count) | Notifications (§15) |
| Emergency card | `Lock-screen access ›` — red-tinted card (`oklch(0.96 0.02 25)` bg, `oklch(0.88 0.05 25)` border, red dot, title in `oklch(0.38 0.13 25)`) | Emergency card (§16) |
| Security & sign-in | `Face ID + passkey ›` (reflects enrolled methods) | Security settings ([06-identity-auth.md](06-identity-auth.md)) |
| Language | `English ›` (current language) | Language picker (English/Hindi/Marathi) |

The Security & sign-in and Language rows are inert in the prototype — no target screens are built; their destinations above are Proposed.

### Acceptance criteria
- The Access & consent and Notifications hints show live counts consistent with those screens.
- The allergy value is rendered in the alert red; the ABHA linked state matches the identity service.

---

## 13. Access & consent

Push screen from Profile or the Home banner; `‹ Back` returns. Header: Instrument Serif 28 px `Access & consent`; subtitle 13 px: `Who can see your records. Every access is logged.`

**Pending consent request card** — shown only while a request awaits a decision: white card with a 1.5 px ocean border, header row `Consent request` + `NEW` pill (ocean tint), body 12.5 px `Dr. Meera Kulkarni · Dermatology, SkinCare Clinic requests access to your records`, and a full-width ink CTA `Review request` → Consent request screen (§14). The card disappears permanently once the request is approved or denied.

**Grants list** — one card per provider/integration with access history:

| Grantee | Sub-line | Status pill | Expiry line | Action |
|---|---|---|---|---|
| Dr. Meera Kulkarni (only after approval) | SkinCare Clinic · Dermatology | Active | `Expires in {chosen duration}` | `Revoke` (red) |
| Dr. Rohan Menon | City General · Endocrinology | Active | Expires 12 Sep 2026 | `Revoke` (red) |
| Apollo Diagnostics | Integration · sends reports only | Active | Ongoing | `Manage` (ocean) |
| Dr. A. Fernandes | City General · Surgery | Revoked | Revoked 20 Nov 2025 | `Re-grant` (ocean) |

Semantics (full model in [07-consent-and-access-control.md](07-consent-and-access-control.md)):

- `Revoke` immediately ends the grant; the doctor loses read access and the row flips to Revoked with a revocation date. Proposed: a confirmation sheet precedes revocation, and a toast `Access revoked · {name} notified` confirms it.
- `Re-grant` reopens the consent flow (§14) pre-filled for that requester.
- Integration rows (write-only report sources) expose `Manage` rather than Revoke — they send data in and hold no read access; management options are specified in [07-consent-and-access-control.md](07-consent-and-access-control.md).
- A doctor-side banner mirrors expiry (the doctor home shows "Access expires — Priya Sharma consent ends 12 Sep 2026", see [03-doctor-app.md](03-doctor-app.md)); dates must agree across both apps.

**Recent access log** — tinted panel `oklch(0.94 0.02 235)`, 12 px / 1.5, newest first, format `{date} {time} — {actor} · {action}`. Demo content:

```text
Recent access log
21 Aug 9:03 — Apollo Diagnostics · report sync
12 Aug 10:41 — Dr. R. Menon · viewed via QR
12 Aug 10:40 — QR scanned · access granted (30 days)
```

The log is a patient-readable projection of the audit trail (PRD §13): QR scans, record views, grants, revocations, report syncs, emergency-card opens. Proposed: the panel shows the latest 3–5 entries with a "View full log" affordance.

### Acceptance criteria
- The pending card appears if and only if an undecided consent request exists, and its CTA opens the consent flow.
- Revoking a grant takes effect immediately and is reflected in both the list and the access log.
- Every QR scan, record view, and grant change performed against this patient appears in the access log with actor and timestamp.

---

## 14. Consent request flow

Push screen from the pending card or a notification CTA; `‹ Back` returns to Access & consent without deciding. Header: Instrument Serif 28 px `Consent request`.

**Requester card**: initials avatar (`MK`), name 14.5 px semibold `Dr. Meera Kulkarni`, credential line 12 px `Dermatology · SkinCare Clinic · verified professional`. The "verified professional" tag is asserted by Atlas's professional-identity verification, not self-declared ([06-identity-auth.md](06-identity-auth.md)).

**Access duration**: section label `ACCESS DURATION` (12 px semibold muted), three pill options `24 hours` / `30 days` / `90 days` — single-select, default **30 days**; selected pill is ink/white.

**Category toggles**: section label `WHAT SHE CAN SEE`; white card of switch rows (40 × 24 px track, white 18 px knob; on = ocean `oklch(0.52 0.13 235)`, off = `rgba(34,48,60,0.25)`):

| Category | Default | Note |
|---|---|---|
| General history | On | |
| Reports & labs | On | |
| Medications | On | |
| Mental health | **Off** | Sub-label under the row title: `Off by default — sensitive category` |

Categories correspond to the information-class policies of PRD §12; the mental-health default-off is a hard product rule (sensitive category), not a per-tenant setting.

**Explainer**, 12 px muted: `She sees only the categories you enable. Access expires automatically and every view is logged in your access log.` (Proposed: pronoun agrees with the requester.)

**Decision buttons**:

- `Approve access` (ink primary) → creates a grant for the selected duration and categories, navigates back to Access & consent where the new grant appears at the top of the list (`Active`, `Expires in {duration}`, `Revoke`), and shows the toast `Access granted · {duration} · every view is logged`. The requester is notified and can now open the record within the granted categories.
- `Deny` (outline secondary) → records the denial, navigates back to Access & consent (no grant row is added, the pending card is gone), and shows the toast `Request denied · requester notified`.

Both outcomes remove the request from its Notifications entry (the `Review request` chip disappears, §15) and write grant/deny events to the audit trail.

Edge cases — Proposed: a request already decided elsewhere (another device) opens in a read-only "already handled" state; an expired request shows "This request has expired" with no decision buttons; approving with all categories off is blocked with an inline hint to enable at least one category.

### Acceptance criteria
- Default state on open: 30 days selected; General/Reports/Medications on; Mental health off with its explanatory sub-label.
- Approve creates a grant matching exactly the chosen duration and enabled categories; the doctor can subsequently see only those categories ([07-consent-and-access-control.md](07-consent-and-access-control.md)).
- Deny produces no grant and notifies the requester; both paths toast the exact strings above and land on Access & consent.

---

## 15. Notifications

Push screen; `‹ Back` returns to Home. Header: Instrument Serif 28 px `Notifications`. List of white cards: status dot (9 px — ocean `oklch(0.52 0.13 235)` for new/actionable, `rgba(34,48,60,0.25)` for read/informational), title 13.5 px semibold, sub-line 12 px muted, optional action chips. Demo list (prototype order — the actionable consent request is pinned above the newer 9:12 report notification; Proposed ordering rule: actionable items first, then newest first):

| Title | Sub-line | Dot | Actions |
|---|---|---|---|
| Dr. Meera Kulkarni requests access | SkinCare Clinic · consent request · 8:40 today | New | `Review request` (ink chip) → Consent request screen; chip shown only while undecided |
| Lipid panel report is ready | Apollo Diagnostics · 9:12 today | New | — (Proposed: tap opens report detail) |
| Dr. R. Menon viewed your records | via QR scan · 12 Aug | Read | — |
| Follow-up with Dr. Menon on 12 Sep | City General · reminder | Read | `Confirm` (ink chip), `Reschedule` (outline chip) |
| Access granted to Dr. R. Menon | 30 days · 12 Aug | Read | — |

Notification types covered (PRD §16): consent request, new report available, records accessed by a doctor, follow-up reminder, access granted. Also produced but not in the demo list — Proposed: access revoked, new prescription, new medical record added. Proposed: per-type notification preferences (PRD §16 requires notifications to be configurable).

Follow-up actions:

- `Confirm` → toast `Follow-up confirmed · added to your calendar`. Proposed: writes a calendar entry via the platform calendar and marks the reminder confirmed (chips replaced by a "Confirmed" state).
- `Reschedule` → toast `Reschedule request sent to City General`. Proposed: sends a reschedule request to the organization; chips replaced by a "Requested" state pending the clinic's response.

Content rule (PRD §16): notification titles/sub-lines identify the event and actor but avoid clinical detail beyond the record name — no result values in notification text.

### Acceptance criteria
- The consent-request notification's `Review request` chip opens the consent flow and disappears once the request is decided.
- Confirm/Reschedule fire exactly one action each, show the specified toasts, and cannot be re-triggered afterward (Proposed).
- Unread state is reflected consistently here, on the Home bell badge, and on the Profile "Notifications" hint.

---

## 16. Emergency card

Push screen from Profile; `‹ Back` returns. Also surfaced **without sign-in** from the device lock screen (approved requirement: offline lock-screen emergency card). Header: Instrument Serif 28 px `Emergency card`; subtitle 13 px: `Visible from the lock screen — no sign-in needed.`

**Card**: white, 16 px radius, 1.5 px red-tinted border `oklch(0.8 0.09 25)`. Header band: solid red `oklch(0.55 0.18 25)`, white text — left `MEDICAL EMERGENCY INFO` (14 px semibold), right `Atlas · ATL-82X92K` (11 px, 85% opacity). Field rows:

| Field | Demo value | Treatment |
|---|---|---|
| Name | Priya Sharma · 34 F | semibold |
| Blood group | B+ | bold, alert red `oklch(0.45 0.15 25)` |
| Allergies | Penicillin — severe | bold, alert red |
| Conditions | Type 2 diabetes | semibold |
| Medications | Metformin · Atorvastatin | semibold |
| Emergency contact | Aman S. · +91 98111 20844 | semibold |

**Logging note**: tinted panel `oklch(0.94 0.02 235)`, 12 px / 1.5: `Anyone who opens this card is logged. Doctors using emergency (break-glass) access are reviewed by compliance.`

Behavior:

- The card's data is cached on-device so it renders offline and from the lock screen (Proposed mechanics: iOS lock-screen widget / Medical-ID-style surface and Android lock-screen shortcut; exact platform integration per [13-non-functional.md](13-non-functional.md)).
- Every open of the card — in-app or from the lock screen — writes an audit event; lock-screen opens are logged when the device next connects (Proposed) and appear in the patient's access log (§13).
- The card shows exactly these six fields; it contains no QR-resolvable token and no records beyond the safety-critical minimum. Doctor-side emergency break-glass access is a separate flow specified in [03-doctor-app.md](03-doctor-app.md) and [07-consent-and-access-control.md](07-consent-and-access-control.md); this card is its patient-side, read-only complement.
- Content derives from the profile and active records; the phone-number and contact fields are patient-editable via Profile (§12).

### Acceptance criteria
- The card renders fully with the device offline and without authentication from the lock-screen entry point.
- Blood group and allergies render in the alert red treatment.
- Card opens generate audit-log entries visible in Access & consent.

---

## 17. Cross-cutting behavior

### 17.1 Toasts as audit feedback

Every consequential action confirms itself with a toast that names the action **and its audit consequence**, reinforcing the platform promise that everything is logged. Toast presentation: dark ink `#22303c` bar, white 13 px text, 12 px radius, anchored 36 px above the bottom edge with 22 px side margins, shadow `0 8px 24px rgba(34,48,60,0.35)`; enters with a 0.25 s slide-up/fade (`translateY(8px)` → 0) and auto-dismisses after 2.8 s; a new toast replaces the current one.

Patient-app toast catalog (exact strings):

| Trigger | Toast |
|---|---|
| Consent approved | `Access granted · {duration} · every view is logged` |
| Consent denied | `Request denied · requester notified` |
| Follow-up confirmed | `Follow-up confirmed · added to your calendar` |
| Follow-up reschedule requested | `Reschedule request sent to City General` |
| Family profile switched (Proposed product string) | `Now viewing {full name}` |
| Grant revoked (Proposed) | `Access revoked · {name} notified` |

### 17.2 Loading, empty, and error states — Proposed

The prototype shows populated states only. Product requirements:

- **Loading**: content areas show lightweight skeleton placeholders in the card geometry of the loaded state (no spinners over blank screens). Home sections load independently; the AI Summary card may resolve later than the rest without blocking.
- **Empty**: first-run patients see the Home structure with the QR card and access banner (`0 providers…` hidden until ≥1 grant exists — banner suppressed at zero), no AI Summary card (hidden until records exist, per [05-ai-features.md](05-ai-features.md) §2.5), an empty medication section ("No active medications"), and empty-state copy in Timeline/Records per §8–§9.
- **Errors**: failed loads show an inline retry row in place of the section; destructive/consequential actions (revoke, deny) that fail show an error toast and leave state unchanged.
- **Offline**: QR present degrades per §7; the emergency card always works offline; other screens show cached data with a subtle staleness indicator.

### 17.3 Localization

All UI strings, dates, and notification texts are localized in English, Hindi, and Marathi. The strings quoted in this spec are the English catalog; clinical record content (report names, doctor notes) renders in its source language. Language is switchable at Welcome (§3) and Profile → Language (§12).

### 17.4 Privacy defaults

- The QR and Atlas ID are identity, not authorization: possessing them grants nothing without the doctor-side auth + consent checks (PRD §33).
- Mental-health category is off by default in every consent grant (§14).
- Notifications never carry clinical values (§15).
- App switcher/screenshot protection for record screens: Proposed, per platform capability, decided in [13-non-functional.md](13-non-functional.md).

### Acceptance criteria
- Every toast in the catalog fires on its trigger with the exact string (parameterized values interpolated).
- No populated screen has an unhandled empty or error rendering in production builds.
- Switching language re-renders all chrome strings without restart.
