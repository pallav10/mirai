# Meridian design system

Purpose: define the complete visual language, tokens, and component library ("1a Meridian") that every Atlas surface — patient app, doctor app, admin app — must implement, so screens built by different teams are pixel-consistent with the approved prototype.

Sources: `design/Atlas Directions.dc.html` (three explored directions; 1a chosen), `design/Atlas App.dc.html` (hi-fi prototype — authoritative for every color, size, spacing, and copy value below), `design/ios-frame.jsx`, `design/android-frame.jsx` (device-frame behavior). Cross-references: [02-patient-app.md](02-patient-app.md), [03-doctor-app.md](03-doctor-app.md), [04-admin-app.md](04-admin-app.md), [05-ai-features.md](05-ai-features.md), [08-qr-subsystem.md](08-qr-subsystem.md).

---

## 1. Design principles

Meridian is "editorial calm": a health record should read like a well-set medical journal, not a dashboard. The principles below are derived from the chosen direction and applied consistently through the prototype.

1. **Calm over dense.** Warm paper backgrounds, generous 22px screen margins, hairline separators instead of boxes-within-boxes. Color is reserved for meaning (status, alerts, actions), never decoration.
2. **Editorial hierarchy.** Serif display type (Instrument Serif) for screen titles and section headings; a quiet humanist sans (Instrument Sans) for everything else; monospace only for machine identifiers (Atlas ID, ABHA address).
3. **Clinical trust.** Safety-relevant information (allergies, emergency data, break-glass) always uses the red hue-25 family and is visually louder than anything else on screen. Provenance, audit, and consent copy is always present in muted ink, never hidden.
4. **One accent.** A single ocean-blue accent `oklch(0.52 0.13 235)` carries brand, links, selection, and the compass logo. Other hues appear only as semantic status colors or timeline category tints.
5. **Dark means focused or dangerous.** Full-dark screens are used only for the QR presentation, doctor sign-in, camera scan, and break-glass — moments of identity, capture, or exception.

### Acceptance criteria
- No screen introduces a hue outside the token tables in §5 without a design-system change.
- Serif type never appears below 19px; sans body text never renders in Instrument Serif.

---

## 2. Directions explored and decision

Three visual directions were produced for the same artifact set (logo + patient home). Direction **1a Meridian** was chosen; 1b and 1c are documented for the record only and must not leak into the product.

| ID | Name | Character | Type | Palette | Signature elements |
|----|------|-----------|------|---------|--------------------|
| **1a** | **Meridian (chosen)** | Editorial calm | Instrument Serif display + Instrument Sans body | Warm paper `#faf7f1`, ink `#22303c`, ocean blue `oklch(0.52 0.13 235)` | Compass-ring logo, serif greetings, hairline lists, left-rail timeline, raised ink QR tab button |
| 1b | Pulse Glass | Soft dark, glassmorphic | Sora headings + Source Sans 3 body | Dark teal gradient `oklch(0.23 0.02 210)→oklch(0.18 0.015 230)`, mint `oklch(0.85 0.13 165)` | Frosted `rgba(255,255,255,0.06)` cards, glowing mint QR tile, floating pill nav |
| 1c | Orbit | Friendly geometric | Outfit (all weights) | Cool white `oklch(0.97 0.008 270)`, indigo `oklch(0.45 0.17 275)` | Bento grid, 24px radii, rotated-diamond logo, emoji greeting |

---

## 3. Logo — compass ring

The mark is an open compass ring with a center dot, drawn as SVG. It must be constructed, not exported as a bitmap, so it scales and recolors cleanly.

```
viewBox: 0 0 40 40
ring:  <circle cx="20" cy="20" r="15" fill="none"
         stroke="{accent}" stroke-width="3.5"
         stroke-dasharray="70 24" stroke-linecap="round"
         transform="rotate(-55 20 20)"/>
dot:   <circle cx="20" cy="20" r="4.5" fill="{accent}"/>
```

- The `70 24` dasharray on a radius-15 circle (circumference ≈ 94.2) leaves one rounded-cap gap of roughly a quarter turn; `rotate(-55°)` positions that opening in the upper-right — this orientation is part of the mark and must not change.
- Stroke color: `oklch(0.52 0.13 235)` on light backgrounds; `oklch(0.72 0.1 235)` on dark/ink backgrounds (doctor login).
- Sizes used: 72px (patient welcome), 56px (doctor login), 40px (brand lockup), 26px (compact header; stroke-width increases to 4 at this size to hold weight).
- Wordmark: "Atlas" in Instrument Serif, ink `#22303c` (26px in the brand lockup, 20px in compact chrome), set to the right of the mark with 10–14px gap.

### Acceptance criteria
- Logo renders identically from a single SVG asset at 26–72px; the ring opening always points upper-right.
- Dark-background usage swaps only the stroke/fill color, never the geometry.

---

## 4. Typography

Font loading (Google Fonts): `Instrument Serif` (regular + italic), `Instrument Sans` weights 400; 500; 600.

> Note: the prototype uses `font-weight:700` in a handful of places (timeline chip letters, FAILED/DEAD_LETTER/RETRYING pills, family-chip avatar initials, the break-glass "!", the emergency-card B+/allergy values at 14px, the compare-table abnormal current value, the AI answer-block question). Proposed: load Instrument Sans 700 as well, or normalize those usages to 600 — do not rely on faux-bold.

### 4.1 Display scale — Instrument Serif

All display sizes actually used in the prototype (px). Line-height 1.1–1.15 for ≥28px, default otherwise.

| Token | Size | Usage (exact instances) |
|-------|------|-------------------------|
| `display-38` | 38 | Patient welcome hero "Your health, in one place" (line-height 1.1) |
| `display-32` | 32 | Doctor login "Atlas for clinicians" (line-height 1.15) |
| `display-30` | 30 | Patient home greeting "Good morning, Priya"; break-glass "Emergency access" (line-height 1.15) |
| `display-28` | 28 | Screen titles: Enter the code, Timeline, Records, Access & consent, Consent request, Notifications, Emergency card |
| `display-26` | 26 | QR-present patient name; Report detail; Hospitalization; Doctor home name; Admin "Organization"; Add to record; "Legacy HIS — events" |
| `display-24` | 24 | Patient profile name |
| `display-22` | 22 | Doctor patient-360 header name; admin dashboard stat values (42 · 12,408 · 3) |
| `display-19` | 19 | In-page section headings: Today's medication, Recent, Recent patients, Integrations |

> The 34px size in the iOS large-title bar belongs to the system device frame (SF Pro 34/700, `ios-frame.jsx`), not to the Meridian ramp; Atlas screens draw their own serif titles instead of the frame's nav bar.

### 4.2 Body scale — Instrument Sans

| Token | Size | Weight | Usage |
|-------|------|--------|-------|
| `body-16` | 16 | 600 | Doctor hero action title ("Scan patient QR") |
| `body-15` | 15 | 400/600 | Primary button labels (600); card titles ("My Atlas QR", 600); auth input text (400 — welcome and doctor-login inputs) |
| `body-14.5` | 14.5 | 400/600 | Welcome subtitle; form input text (add-record, break-glass); consent requester name; "Simulate scan" label |
| `body-14` | 14 | 600 | List item titles, row values, back links (400), scan-screen caption (400) |
| `body-13.5` | 13.5 | 400/600 | Table/row values, AI summary body (patient), key-value values, secondary buttons, dark-screen subtitles/body (doctor-login "City General Hospital · SSO", break-glass intro) |
| `body-13` | 13 | 400 | Metadata lines, subtitles, toast text, row labels |
| `body-12.5` | 12.5 | 400/600 | Links ("View report", "All"), chips, descriptions |
| `body-12` | 12 | 400/600 | Field labels, filter pills, secondary metadata, info panels |
| `body-11.5` | 11.5 | 400/600 | Fine metadata, AI chips, language pills; break-glass `REASON` overline (600, `letter-spacing:0.05em`) |
| `body-11` | 11 | 400/600 | Overlines/section labels (600, uppercase, `letter-spacing:0.04–0.08em`); metadata lines and hints (400 — timeline card metadata, admin stat labels, search hint, session pill, "Off by default" note) |
| `body-10.5` | 10.5 | 400/600 | Tab bar labels, status pills, AI disclaimers, timestamps |
| `body-10` | 10 | 400/600/700 | PDF chips (600), FAILED/DEAD_LETTER pills (700), family-chip avatar initials (700), trend period labels (400) |
| `body-9.5` | 9.5 | 700 | Timeline event-type chip letters (`letter-spacing:0.03em`) |

### 4.3 Monospace

`ui-monospace, Menlo, monospace` — machine identifiers only: Atlas ID `ATL-82X92K` (12px on cards, 14px on the QR screen, 12.5px on profile), ABHA address `priya@abdm` (13px).

### Acceptance criteria
- Every text style on every screen maps to exactly one row of the tables above.
- Overline labels (11px/600; 11.5px/600 with 0.05em on the break-glass dark screen) are uppercase with 0.04–0.08em tracking; timeline chip letters use 0.03em; no other text is letter-spaced.

---

## 5. Color tokens

All values are verbatim from the prototype. OKLCH is the source of truth; ship OKLCH where supported and pre-computed sRGB fallbacks elsewhere.

### 5.1 Core palette

| Token | Value | Usage |
|-------|-------|-------|
| `canvas` | `#efece5` | Page/canvas ground behind device frames (design-tool chrome; web shell background) |
| `paper` | `#faf7f1` | Default screen background |
| `paper-raised` | `#fdfbf7` | Tab bar, sticky headers, action-bar footers |
| `surface` | `#ffffff` | Cards, inputs, list rows |
| `ink` | `#22303c` | Primary text, primary buttons, dark QR screen, toast |
| `accent` | `oklch(0.52 0.13 235)` | Brand accent: logo, selected toggle track, active dots, timeline med-dot, trend "now" bar |
| `accent-text` | `oklch(0.45 0.13 235)` | Links, active tab-bar item, tab underline; hover `oklch(0.38 0.12 235)` |
| `accent-deep` | `oklch(0.42 0.12 235)` | Text on accent tints (NEW/Amended pills, avatar initials, timeline PDF pills). The square PDF doc-icon tiles use `accent-text` `oklch(0.45 0.13 235)` instead |
| `accent-chip-text` | `oklch(0.4 0.11 235)` | Condition chip text on doctor overview ("Type 2 diabetes · 2022", "Allergic rhinitis") |
| `accent-tint` | `oklch(0.94 0.02 235)` | Info panels, PDF chips, session pill, NEW pill, access banner |
| `accent-tint-strong` | `oklch(0.9 0.03 235)` | Avatar circles (PS, MK, RM, CG) |
| `accent-tint-muted` | `oklch(0.93 0.01 235)` | De-emphasized avatars (non-highlighted roster patients; text `rgba(34,48,60,0.55)`) |
| `accent-rail` | `oklch(0.85 0.05 235)` | Recent-timeline left rail; oldest trend bar |
| `accent-mid` | `oklch(0.72 0.09 235)` | Middle trend bar |
| `accent-underline` | `oklch(0.72 0.08 235)` | Underline under "Present to doctor"; Dictate chip border |
| `table-head` | `oklch(0.96 0.01 235)` | Compare-table header row background |

### 5.2 Ink alpha scale (on light surfaces)

All derived from ink `#22303c` → `rgba(34,48,60,α)`:

| α | Usage |
|---|-------|
| 0.05 | Card shadow `0 1px 2px` |
| 0.08 | Hairline row separators; neutral pill bg |
| 0.10 | Card borders; tab-bar top border; DEAD_LETTER pill bg |
| 0.12 | QR tile border; chrome borders |
| 0.15 | Filter-pill borders; input borders (doctor search) |
| 0.18 | Form input borders; secondary button borders |
| 0.20 | Secondary button borders (alt); OTP idle cell border |
| 0.25 | Toggle-off track; muted notification dots; dashed upload border |
| 0.30 | Inactive tab-bar dots (directions comp) |
| 0.35 | Unfilled med dot ring; OTP placeholder dot |
| 0.45 | Timestamps, chevrons, disclaimers, timeline year-group labels |
| 0.50 | Secondary text, inactive tab labels, timeline card metadata |
| 0.55 | Subtitles, metadata, field labels |
| 0.60 | Back links, muted values |
| 0.65 | DEAD_LETTER pill text |
| 0.70 | Info-panel body text |

### 5.3 Semantic colors

**Danger / emergency (hue 25):**

| Token | Value | Usage |
|-------|-------|-------|
| `danger` | `oklch(0.55 0.18 25)` | Notification badge dot, emergency-card header bar, break-glass icon & button, selected reason chip, emergency banner |
| `danger-text` | `oklch(0.45 0.15 25)` | FAILED pill text, B+ / allergy values on emergency card |
| `danger-text-soft` | `oklch(0.45 0.12 25)` | "Lock-screen access ›" trailing label on the profile emergency row |
| `danger-action` | `oklch(0.5 0.16 25)` | "Revoke" link; "Penicillin" allergy value on patient profile |
| `danger-deep` | `oklch(0.42 0.15 25)` / `oklch(0.4 0.15 25)` / `oklch(0.38 0.13 25)` | Allergy banner text; allergy-conflict body; "Emergency card" row label |
| `danger-tint` | `oklch(0.96 0.02 25)` / `oklch(0.95 0.03 25)` / `oklch(0.94 0.04 25)` | Emergency row bg; allergy banner & conflict bg; FAILED pill bg |
| `danger-border` | `oklch(0.88 0.05 25)` / `oklch(0.85 0.06 25)` / `oklch(0.8 0.09 25)` / `oklch(0.7 0.14 25)` | Emergency row border; allergy banner border; emergency-card frame; conflict/queue alert borders |
| `danger-on-dark` | `oklch(0.75 0.12 25)` text, `oklch(0.55 0.1 25)` underline | "Emergency access — no QR available" link on scan screen |

**Success (hue 150):** `oklch(0.65 0.15 150)` integration-healthy dot · `oklch(0.8 0.15 150)` live secure-token dot (dark bg) · `oklch(0.93 0.05 150)` / `oklch(0.42 0.12 150)` Final/Active/Linked pill bg/text · `oklch(0.5 0.12 150)` inline "Final", "Improving", "Stable, recovered".

**Warning (hue 60–80):** `oklch(0.55 0.15 60)` out-of-range values "168 mg/dL ▲" · `oklch(0.93 0.06 80)` / `oklch(0.45 0.12 60)` Preliminary/RETRYING pill bg/text.

**AI (hue 245–260):** see §7.10.

### 5.4 Dark screens

| Token | Value | Screens |
|-------|-------|---------|
| `dark-ink` | `#22303c` | Patient QR-present; doctor login |
| `dark-scan` | `#101820` | Doctor QR scan |
| `dark-emergency` | `#1c1214` | Break-glass emergency access |
| `accent-on-dark` | `oklch(0.72 0.1 235)` | Logo stroke, scan-frame corners, scan reticle |
| `accent-btn-dark` | `oklch(0.62 0.12 235)` | Primary buttons on dark ("Sign in with MFA", "Simulate scan") |
| `accent-caption-dark` | `oklch(0.8 0.1 235)` | Scan progress caption |

White-alpha scale on dark: text `rgba(255,255,255,0.7 / 0.65 / 0.6 / 0.55 / 0.5 / 0.45)`; input bg `rgba(255,255,255,0.08)`; chip bg `rgba(255,255,255,0.1)`; borders `rgba(255,255,255,0.25)` (inputs) and `rgba(255,255,255,0.2)` (reason chips); scan-reticle fill `rgba(255,255,255,0.04)`.

### Acceptance criteria
- No raw hex/oklch literal in app code — every color resolves through a named token from §5.
- Ink-alpha values are generated from the single ink token, not hardcoded grays.

---

## 6. Layout, spacing, radii

| Rule | Value |
|------|-------|
| Screen horizontal margin | 22px (list/content screens); 28px (auth, QR-present, scan, break-glass) |
| Content top offset | 62px from screen top (clears status bar / dynamic island); titles then stack 12–14px below the back link (24px on the OTP screen) |
| Card gap in lists | 8px |
| Section vertical rhythm | 14–20px between blocks (`margin: 14/16/18/20px 22px 0`) |
| Row padding (key-value lists) | 10–12px vertical (11px profile/consent, 10px emergency card, 12px report table), hairline `rgba(34,48,60,0.08)` between rows; card padding `4px 16px` (emergency card `6px 16px`) |
| Card radius | **14px** list/info cards · **16px** feature cards (QR card, report table, emergency card, AI patient card) |
| Control radius | 12px buttons & inputs · 10px nested tiles/buttons · 999px pills & toggles · 6px PDF doc-icon tile |
| Card border | `1px solid rgba(34,48,60,0.1)`; emphasis variant `1.5px` accent (pending consent) or danger (Legacy HIS, allergy conflict) |
| Card shadow | `0 1px 2px rgba(34,48,60,0.05)` (QR hero card only; other cards are flat) |

---

## 7. Component library

### 7.1 Buttons

| Variant | Spec | States |
|---------|------|--------|
| **Primary (ink)** | bg `#22303c`, text `#fff`, radius 12, padding `15px 0` (full-width large; 12–14px medium), 15px/600 (13.5–14px/600 medium) | Hover/press: `opacity:0.88`. Examples: "Continue with OTP", "Verify & continue", "Approve access", "+ Add to record", "Save to Priya's record", "Retry all recoverable", "Share" |
| **Secondary (outline)** | bg `#fff`, border `1px rgba(34,48,60,0.18–0.2)`, ink text, radius 12, 13.5–14px/600 | Hover/press: bg `#f2eee6`. Examples: "Deny", "End consult", "Download PDF" |
| **Primary on dark** | bg `oklch(0.62 0.12 235)`, white text, radius 12 (or 999 for "Simulate scan") | Hover: `opacity:0.88` |
| **Destructive / emergency** | bg `oklch(0.55 0.18 25)`, white text, radius 12 | "Proceed — logged & reviewed"; hover `opacity:0.88` |
| **Small pill button** | radius 999, padding `6–7px 14px`, 12px/600; filled ink ("Confirm", "Review request", "Resolve identity") or white + `1px rgba(34,48,60,0.18)` border ("Reschedule", "View payload") | — |
| **Text link** | `oklch(0.45 0.13 235)`, 12.5px | Hover `oklch(0.38 0.12 235)` |
| **Underlined action link** | 12.5px/600 accent-text with `border-bottom: 1px oklch(0.72 0.08 235)` | "Present to doctor" |

Proposed: pressed state = same 0.88 opacity plus scale 0.98; disabled = 40% opacity, non-interactive (prototype shows no disabled buttons).

### 7.2 Pill toggles and filter chips

Single-select chip groups (timeline filters All/Consultations/Reports/Hospital; records tabs Reports/Prescriptions/Discharge; duration picker 24 hours/30 days/90 days; add-record type Diagnosis/Prescription/Report; report List/Compare; language English/हिंदी/मराठी; family switcher):

- Unselected: bg `#fff`, text ink, border `1px rgba(34,48,60,0.15)`, radius 999, 12–12.5px/600, padding `5px 12px` (filters) or `6px 14px` (segments).
- **Selected: bg `#22303c`, text `#fff`** (border retained). Exactly one selected per group.
- Language pills (welcome screen) are compact: 11.5px, padding `4px 12px`; selected 600, unselected weight 400.
- Family chips add a 26px initials avatar (avatar bg `rgba(255,255,255,0.2)` when selected, `oklch(0.9 0.03 235)` otherwise).
- Break-glass reason chips are the dark-surface variant: 12.5px, padding `7px 14px`; selected bg `oklch(0.55 0.18 25)` at weight 600; unselected `rgba(255,255,255,0.1)` + `1px rgba(255,255,255,0.2)` border at weight 400.

### 7.3 Underline tabs (doctor patient-360)

Overview · Timeline · Reports · Meds: 12.5px/600, padding `9px 12px`; active = ink text + `2px` bottom border `oklch(0.45 0.13 235)`; inactive = `rgba(34,48,60,0.5)` + transparent border.

### 7.4 Cards and panels

| Kind | Spec |
|------|------|
| List card | `#fff`, radius 14, border `1px rgba(34,48,60,0.1)`, padding `12px 14px` (timeline cards) / `13px 15px` (report, roster, notification cards) / `14px 16px` (settings and consent rows), internal gap 12px, optional trailing chevron `›` in `rgba(34,48,60,0.45)` |
| Feature card | `#fff`, radius 16, padding 16px (QR hero also gets the §6 shadow) |
| Key-value card | radius 16, padding `4px 16px`; rows label 13px `rgba(34,48,60,0.55)` / value 13.5px 600, hairline separators |
| Info panel (blue) | bg `oklch(0.94 0.02 235)`, radius 14, padding `12px 16px`, 12px text `rgba(34,48,60,0.7)`, line-height 1.5 — provenance, access log, expiry notices, emergency-card audit note |
| Alert panel (red) | bg `oklch(0.95 0.03 25)`, border `1px oklch(0.85 0.06 25)` (allergy banner) or `1.5px oklch(0.7 0.14 25)` (allergy conflict), radius 10–12 |
| Emphasis card | standard card with `1.5px` accent border (pending consent request) or `1.5px oklch(0.7 0.14 25)` (failing integration) |
| Upload dropzone | `1.5px dashed rgba(34,48,60,0.25)`, radius 14, padding 26px, centered, `#fff` |

### 7.5 Status pills

Shape: radius 999, padding `3px 9px`, 10.5px/600 (10px/700 for FAILED/DEAD_LETTER/RETRYING in the admin queue). Exact mapping:

| Status | Background | Text |
|--------|------------|------|
| `Final`, `Active`, `Linked` | `oklch(0.93 0.05 150)` | `oklch(0.42 0.12 150)` |
| `Preliminary`, `RETRYING` | `oklch(0.93 0.06 80)` | `oklch(0.45 0.12 60)` |
| `Amended`, `NEW` | `oklch(0.94 0.02 235)` | `oklch(0.42 0.12 235)` |
| `Revoked`, `FAILED` | `oklch(0.94 0.04 25)` | `oklch(0.45 0.15 25)` |
| `Done` (default/neutral) | `rgba(34,48,60,0.08)` | `rgba(34,48,60,0.6)` |
| `DEAD_LETTER` | `rgba(34,48,60,0.1)` | `rgba(34,48,60,0.65)` |

Report lifecycle uses Final/Preliminary/Amended; prescriptions Active/Done; consent grants Active/Revoked; integration events FAILED/RETRYING/DEAD_LETTER.

Exception: the ABHA "Linked" pill on the patient profile renders compact — 10px/600, padding `2px 8px` — with the Final/Active colors.

### 7.6 Timeline event-type badges

Square badge 38×38, radius 12, containing a 3-letter category code at 9.5px/700, `letter-spacing:0.03em`. Colors derive from one hue per category via the tint formula:

```
badge background: oklch(0.93 0.045 h)
badge text:       oklch(0.4  0.12  h)
```

| Code | Category | Hue h |
|------|----------|-------|
| LAB | Lab report | 235 |
| CON | Consultation | 280 |
| RX | Prescription | 150 |
| VAX | Vaccination | 190 |
| HSP | Hospitalization | 60 |
| DX | Diagnosis | 25 |
| DOC | Uploaded document | 235 |

Timeline cards: standard list card; metadata line 11px (`{date} · {type}`), title 14px/600, org 12px `rgba(34,48,60,0.55)`; events with an attachment show a `PDF` pill (10px/600, accent-tint bg, accent-deep text). Year group headers: 11px/600, `letter-spacing:0.08em`, `rgba(34,48,60,0.45)`, with a 1px `rgba(34,48,60,0.1)` rule filling the remaining width.

### 7.7 Toggle switch

40×24 track, radius 999; knob 18px white circle at `top:3px`, `left:3px` (off) → `left:19px` (on), `transition: left 0.15s`. Track on = `oklch(0.52 0.13 235)`; off = `rgba(34,48,60,0.25)`. Used for consent categories (Mental health defaults off — see [07-consent-and-access-control.md](07-consent-and-access-control.md)).

### 7.8 Form inputs

Text input / textarea: bg `#fff`, border `1px rgba(34,48,60,0.18)` (search: 0.15), radius 12, padding `13–14px 15–16px`, 14–15px text. Dark variant: bg `rgba(255,255,255,0.08)`, border `1px rgba(255,255,255,0.25)`, white text. Field labels above: 12px/600 `rgba(34,48,60,0.55)`, 5px gap. OTP cells: 56×64, radius 12, `#fff`; filled cell border `1.5px oklch(0.52 0.13 235)`, empty `1px rgba(34,48,60,0.2)`, digit 24px/600. Dictate affordance: pill chip `● Dictate`, 11.5px/600 accent-text, border `1px oklch(0.72 0.08 235)`.

### 7.9 Toast

Single bottom toast, used for audit/confirmation feedback ("Access granted · 30 days · every view is logged", "Saved to record · signed & audit logged"):

- Position: absolute within the screen, `left/right: 22px`, `bottom: 36px`, above all content (z-index 50).
- Style: bg ink, white 13px text, radius 12, padding `12px 16px`, shadow `0 8px 24px rgba(34,48,60,0.35)`.
- Entry animation `0.25s ease`: fade from 0 and rise 8px (`translateY(8px) → none`).
- Auto-dismisses after **2.8s**; a new toast replaces the current one immediately (timer resets). No exit animation specified — Proposed: 0.2s fade-out.

### 7.10 AI summary card

Shared treatment for the patient home summary and the doctor clinical brief (see [05-ai-features.md](05-ai-features.md)):

- Container: `background: linear-gradient(135deg, oklch(0.96 0.02 235), oklch(0.94 0.03 260))`, border `1px oklch(0.86 0.05 245)`, radius 16 (patient) / 14 (doctor), padding `15px 16px` / `13px 15px`.
- Header row: 15px spark glyph (rotated-45° rounded square, fill `oklch(0.5 0.14 260)`) + label `AI SUMMARY` / `AI CLINICAL SUMMARY` at 11px/600, `letter-spacing:0.06em`, `oklch(0.42 0.12 260)`; right-aligned meta ("Updated today, 9:15" / "Generating…") 10.5px.
- Body: 13.5px (patient) / 13px (doctor), line-height 1.55, ink; key findings bolded (e.g. **TG 168 ↑**, **Penicillin allergy.**).
- Disclaimer footer, always present, 10.5px `rgba(34,48,60,0.45)`: patient "Generated from 7 records · not medical advice — ask your doctor"; doctor "Generated from 7 records at scan · verify against source data before clinical decisions".
- Generating state (doctor, ~1.6s after scan): three skeleton bars, height 9px, radius 5, widths 95/85/60%, color `oklch(0.88 0.03 250)`.
- Ask-the-record chips: radius 999, 11.5px/600, bg `rgba(255,255,255,0.75)`, border `1px oklch(0.86 0.05 245)`, text `oklch(0.42 0.12 260)`. Tapping renders an answer block inside the card: bg `rgba(255,255,255,0.8)`, radius 10, padding `10px 12px`, question 11px/700 in the AI label color, answer 12.5px/1.5.

### 7.11 Avatars and indicator dots

- Initials avatar: circle (40px header, 46px patient-360, 56px profile, 42px consent, 38px roster, 26px family chip), bg `oklch(0.9 0.03 235)`, initials 600 in `oklch(0.42 0.12 235)`. Admin org avatar uses radius 12 square. De-emphasized roster avatars use `accent-tint-muted`.
- Unread/notification dot: 10px `oklch(0.55 0.18 25)` circle with 2px paper ring, top-right of the bell button.
- Notification list dots: 9px; unread `oklch(0.52 0.13 235)`, read `rgba(34,48,60,0.25)`.
- Integration health dots: 10px; healthy `oklch(0.65 0.15 150)`, failing `oklch(0.55 0.18 25)`.
- Medication state dot: 8px; taken dose = filled `oklch(0.52 0.13 235)`; upcoming dose = unfilled ring `1.5px rgba(34,48,60,0.35)`.

### 7.12 Data display (trend bars, compare table)

- Trend chart (HbA1c): column bars 34px wide, radius `6px 6px 0 0`, height ∝ value, chronological tint ramp `oklch(0.85 0.05 235) → oklch(0.72 0.09 235) → oklch(0.52 0.13 235)` (newest = full accent); value labels 11px/600 above, period labels 10px below; header overline `TREND — HBA1C` + trend word in success/warn color.
- Compare table: grid `1.3fr 1fr 1fr`; header row bg `oklch(0.96 0.01 235)`, 11px/600 uppercase; older column values `rgba(34,48,60,0.6)`; current column 600 with direction glyphs `▲▼`; abnormal current values `oklch(0.55 0.15 60)` at weight 700.

### Acceptance criteria
- Selected chips/pills always render ink-on-white → white-on-ink inversion; no accent-filled selection chips exist (the break-glass reason chips are the sole exception, using the danger fill per §7.2).
- Status pill colors match §7.5 exactly for all six statuses plus queue states.
- Every AI surface shows its disclaimer footer; the disclaimer is not dismissible.
- Toast timing is 2.8s with 0.25s ease entry on all platforms.

---

## 8. QR visual specification

The rendered QR (see [08-qr-subsystem.md](08-qr-subsystem.md) for payload/rotation semantics) has a fixed visual treatment:

- Grid: 21×21 modules on a `0 0 126 126` viewBox (6-unit modules). Three finder patterns (top-left, top-right, bottom-left) drawn as rounded concentric squares: 7-module outer (rx 3), 5-module inner in background color (rx 2), 3-module core (rx 1.5). Foreground `#22303c` on `#ffffff`.
- Placements: **home card** 84×84, padding 6, border `1px rgba(34,48,60,0.12)`, radius 10 · **present screen** 250×250 white tile, radius 20, padding 18, shadow `0 10px 40px rgba(0,0,0,0.35)` on the ink background · **scan preview** 130×130, radius 12, padding 8, opacity 0.95.
- Present screen furniture: patient name in `display-26` serif, Atlas ID in 14px mono `rgba(255,255,255,0.8)`, and a live-token pill (`rgba(255,255,255,0.1)` bg, radius 999, 12.5px) with a 7px `oklch(0.8 0.15 150)` dot: "Secure token · rotates in 4:52". Footer copy (12.5px, `rgba(255,255,255,0.55)`, max-width 280px, centered): "This code identifies you — it contains no medical data. The doctor sees only what you've authorized."
- Doctor scan reticle: 240×240 area, fill `rgba(255,255,255,0.04)`, radius 24, with four 44×44 corner brackets `3.5px oklch(0.72 0.1 235)` (18px corner radius).

---

## 9. Tab bar (patient app)

Visible on the four top-level patient screens (Home, Timeline, Records, Profile); hidden on all pushed/detail and dark screens.

- Container: bg `#fdfbf7`, `border-top: 1px rgba(34,48,60,0.1)`, padding `12px 8px 30px` (the 30px bottom clears the home indicator / gesture area).
- Items: label 10.5px/600 with a 5px dot centered 4px above; active color `oklch(0.45 0.13 235)` (dot + label), inactive `rgba(34,48,60,0.5)`.
- **Raised center QR button**: 46px circle, bg ink, `margin-top:-18px` so it overhangs the bar, shadow `0 4px 10px rgba(34,48,60,0.3)`; glyph = 18px square outline `2.5px #fff`, radius 4. Tapping opens the full-screen QR-present screen; it has no label and no active state (it presents modally over the tabs).

### Acceptance criteria
- Exactly five slots in order Home · Timeline · [QR] · Records · Profile; QR is always the raised center element.
- The active tab is indicated by color only (dot + label); there is no background highlight.

---

## 10. Dark screens and status-bar rule

| Screen | Background | Notes |
|--------|------------|-------|
| Patient — QR present | `#22303c` | White QR tile, live-token pill |
| Doctor — login | `#22303c` | Logo stroke `oklch(0.72 0.1 235)`, glass inputs |
| Doctor — QR scan | `#101820` | Camera view; accent corner brackets |
| Doctor — break-glass | `#1c1214` | Red-shifted near-black; danger accents |

**Rule:** whenever one of these screens is frontmost, the OS status bar renders light content (white time/battery/signal) — the prototype drives this via the frame's `dark` flag for exactly these four screens. All other screens use dark status-bar content on paper. Dark screens omit the tab bar and use 28px margins; QR-present, scan, and break-glass carry a top back/close link at the 62px offset (doctor login is the entry screen and has none).

---

## 11. Safe areas and device frames

### 11.1 Shared rules

- Top: all screens begin content at **62px** from the top edge, clearing the iOS dynamic island / status bar and the Android status bar + punch-hole. Nothing interactive sits above 62px.
- Bottom, screens with tab bar: the bar's 30px bottom padding is the safe-area allowance; scrollable content above it needs only 16px bottom padding.
- Bottom, full-bleed dark screens: 34px bottom padding (doctor login and the auth screens: 40px).
- Bottom, action-bar footers (doctor 360): `padding: 12px 22px 30px` on `#fdfbf7` with hairline top border.
- Proposed: implement these as `env(safe-area-inset-*)`/`WindowInsets` with the prototype values as the reference device (390×844) minimums.

### 11.2 iOS frame (`ios-frame.jsx` — iOS 26 "liquid glass")

Reference: 402×874 frame (prototype renders at 390×844), corner radius 48, dynamic island 126×37 at top 11, status bar SF Pro 17/590 showing 9:41 + signal/wifi/battery, home indicator 139×5 pinned 8px from bottom (white at 70% on dark screens, black at 25% on light). System affordances (glass pills, 26px-radius inset lists, glass keyboard) belong to the frame; Atlas draws its own Meridian chrome inside the safe area and does not use the system large-title nav bar.

### 11.3 Android frame (`android-frame.jsx` — Material 3)

Reference: 412×892, 8px bezel border `rgba(116,119,117,0.5)`, corner radius 18, 40px status bar with centered 24px punch-hole and 9:30 time, bottom gesture pill 108×4 at 40% opacity (24px band). The same Atlas screens render unchanged inside the Android frame — the prototype's device switcher swaps only the shell. Meridian intentionally does not restyle to Material defaults (no M3 top app bar, dynamic color, or Roboto); the 62px top offset and 30px bottom padding absorb the Android insets. Dark screens set the gesture pill white.

### Acceptance criteria
- The same screen source layouts render correctly in both frames with no per-platform layout forks beyond inset handling.
- On dark screens both platforms show light status-bar icons and a light gesture/home indicator.

---

## 12. Motion

Only three motions exist in the prototype; keep motion at this restraint level:

| Motion | Spec |
|--------|------|
| Toast entry | `0.25s ease`, fade in + 8px rise; auto-dismiss 2.8s |
| Toggle knob | `left 0.15s` slide |
| AI generating → ready | Skeleton bars swap to content after ~1.6s (see [05-ai-features.md](05-ai-features.md) for real latency handling) |

Scan flow timing: "Simulate scan" shows the recognized QR + caption "Identity resolved · checking consent…" for ~1.1s before navigating to the patient 360. Proposed: screen transitions use platform-default push/modal animations; no custom transitions.

---

## 13. Accessibility

- **Contrast.** Ink `#22303c` on paper `#faf7f1` ≈ 12.6:1; white on ink ≈ 13.5:1; accent-text `oklch(0.45 0.13 235)` on white ≈ 7.0:1 (≈ 6.6:1 on paper) — these primary pairs pass WCAG AA. The low-alpha metadata inks do not: composited on paper, `rgba(34,48,60,0.45)` ≈ 2.6:1, `0.5` ≈ 2.9:1, `0.55` ≈ 3.3:1, `0.6` ≈ 3.8:1 — all below AA 4.5:1 (0.55–0.6 clear only the 3:1 large-text bar); the scale first passes AA at 0.7 (≈ 5.0:1). Proposed: keep alphas ≤ 0.6 for non-essential, duplicated metadata only (timestamps, chevrons, decoration), raise any sole-source text to ≥ 0.7, and reserve alphas ≤ 0.35 for non-text (borders, dots, placeholders).
- **Touch targets.** Minimum 44×44px effective target. Buttons (≥44px tall), inputs, OTP cells, the 46px QR tab button, and 40px avatars/toggles-with-padding comply; small pills (filter chips ~29px, status-adjacent links, "Revoke") must be padded to a 44px hit area without changing their visual size (Proposed).
- **Color independence.** Status is never color-only: pills carry text labels (Final, FAILED), trend arrows use glyphs (▲▼), toggle state is positional, active tabs pair color with the dot/underline. Maintain this for new components.
- **Type floor.** 9.5–10px sizes are limited to badge codes, compact pill labels, and trend axis labels; running body copy never renders below 12px (info panels and card metadata are the 12px floor; 10.5–11.5px is labels, disclaimers, and fine metadata only). Support OS text scaling on body/list styles (Proposed: cap display serif scaling at 1.3× to protect layouts).
- **Language.** All patient-app strings localize to English/Hindi/Marathi (welcome-screen pills: English · हिंदी · मराठी); the doctor app and admin console ship English-only at MVP per [13-non-functional.md](13-non-functional.md) §8.1. Devanagari must render in a matched fallback stack since Instrument Sans lacks it (Proposed: Noto Sans Devanagari, matched x-height).
- **Screen readers.** QR images get a functional label ("Your Atlas QR code, ID ATL-82X92K"), not a description of the pattern; toasts announce politely (aria-live/polite equivalents); the raised QR tab button is labeled "Present my QR".

### Acceptance criteria
- Automated contrast checks pass AA for the primary text pairs (ink, white-on-ink, accent-text, and the semantic pill/text colors on their surfaces); ink alphas below 0.7 used as text are a documented exception tracked against the Proposed remediation above, and no new text style may be added below AA.
- Every interactive element reports a ≥44px hit rect in accessibility audits on both platforms.
- Switching language re-renders all chrome copy without layout truncation on a 390pt-wide device.
