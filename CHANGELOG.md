# Mirai — Change Log

Patient-centric health-record platform · design & prototype history
Last updated: 2 Sep 2026

## 1 · Foundation
- Read the Atlas PRD; scoped patient + doctor apps with hi-fi, clickable screens and realistic sample data.
- Explored visual directions; committed to a calm clinical look (Instrument Serif/Sans, deep ink `#22303c`, blue accents in oklch, warm paper background).
- Built the interactive prototype (`Mirai App.dc.html`) with role switcher: **Patient** (welcome/OTP, home, QR, timeline, records, report detail, profile, access & consent, notifications, emergency card) · **Doctor** (SSO login, home, QR scan, patient 360° with Overview/Timeline/Reports/Meds tabs, add-to-record) · **Admin** (dashboard, integration error queue).

## 2 · AI everywhere (PRD addendum)
- AI health summary card on the patient home; AI clinical summary on the doctor's patient 360°, generated at QR scan with a skeleton "Generating…" state.
- Ask-the-record chips with per-question AI answers.
- Later made the summary **AGUI (generative UI)**: the AI picks visuals per visit — trend line chart + gauge bar (e.g. HbA1c trend + triglycerides), rendered per patient.

## 3 · Approved recommendations (18-point review)
Added: doctor patient search (identity-only until consent), grant/temporary access flow with duration picker, report lifecycle states (Final/Preliminary/Amended), hospitalization/discharge detail, emergency break-glass access with compliance banner, family profiles, per-category consent toggles (mental health off by default), follow-up confirm/reschedule, report compare view, drug-allergy warning on prescribing (with override flag), voice dictation for notes, biometric/passkey login cues, ABHA/ABDM alignment notes.

## 4 · Architecture documents
- **Canvas.dc.html** — high-level multi-tenant system design diagram.
- **Mirai Architecture.dc.html** — 23-section end-to-end architecture (edge/gateway, identity, consent & QR access model, domain services, data, integration, eventing, security, DPDPA/compliance, DR, rollout), incl. **AI layer**: RAG pipeline (authorize → retrieve → generate → verify with citations) and pluggable AI providers (Azure OpenAI, Bedrock, Vertex, vLLM) behind a provider gateway.
- **Mirai LLD.dc.html** — component-level design with formal notation: UML component diagram, two sequence diagrams (QR consult, report sync), activity flowchart (integration ingest), domain class model.

## 5 · Targeted product edits
- Treating doctor's name shown in both AI summaries.
- **Auto-approve doctor requests** — standalone setting on Access & consent, on by default, revocable per grant.
- "Your treating organizations" card — hospitals already hold the records they created; consent governs outside doctors.
- **DPDPA 2023 compliance**: purpose-limited revocable consent copy + data-rights card (download, correction/erasure, nominee, grievance officer).
- OPD throughput stat strip on doctor home (23 patients today · 14s scan→chart · 0 paper files).
- Timeline filter subtitle fixes; enhanced timeline grouping.
- **PDF viewer**: tappable PDF badges in doctor timeline + report rows open a full-screen document viewer (letterhead, results table, audit-logged footer); non-lipid documents render a generic body.

## 6 · Multi-doctor, multi-patient
Doctor login became a 3-profile picker (later 4); each profile sees its own roster, patient, AI brief, AGUI visuals, timeline, reports, meds:
1. **Dr. Rohan Menon** — Endocrinology → Priya Sharma (34 F, T2DM)
2. **Dr. Kavita Rao** — Hepatology → Arjun Mehta (48 M, NAFLD cirrhosis; ALT trend, FibroScan gauge)
3. **Dr. Anita Kapoor** — Oncology → Lakshmi Nair (58 F, breast carcinoma IIA; CA 15-3 trend, chemo 4/6 progress)
4. **Dr. Vikram Bose** — Cardiology → **Ravi Krishnan** (62 M, post-PCI, HF EF 38%) — **shared-care patient**: also opens from Dr. Menon's roster with a consent-scope banner and an endocrine-scoped brief/visuals; records added by either doctor appear for both.

## 7 · Platform & polish
- **Android compatibility** — iOS/Android device toggle; full prototype renders in either frame (also a Tweaks prop).
- Chart label clamping, timeline/report data fixes per design review.

## 8 · Rebrand → Mirai (Mira-Ai)
- New sparkle logo (four-point star + orbiting dot; blue + white variants).
- All app copy Atlas → Mirai; patient IDs ATL- → MIR-.
- Architecture, system-design, LLD and directions docs rebranded too (names, logo, mirai.health domain, mirai_id fields, MIR- IDs); consent section updated with auto-approve + shared-care grants; AI section notes AGUI adaptive visuals (2 Sep 2026).

## Files
- `Mirai App.dc.html` — the Mirai interactive prototype (patient · doctor · admin)
- `Canvas.dc.html` — system design diagram
- `Mirai Architecture.dc.html` — end-to-end architecture doc
- `Mirai LLD.dc.html` — UML component/sequence/activity/class diagrams
- `Mirai Directions.dc.html` — early visual directions
