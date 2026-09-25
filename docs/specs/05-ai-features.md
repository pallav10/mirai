# 05 — AI Features

Specifies every AI-powered capability in Atlas — the patient plain-language summary, the scan-time clinical brief, ask-the-record Q&A, and report explanation — plus the RAG pipeline, the pluggable provider gateway, embeddings and caching, and the safety and evaluation regime that governs all of them.

Sources: `Atlas Architecture.dc.html` (§9 "AI summary" service row, §10 AI architecture in full, §12 async consumers, §19 technology table), `Atlas App.dc.html` (patient home AI card; doctor overview AI clinical summary, generating state, chips, and `aiMap` demo answers; scan and emergency flows), `Canvas.dc.html` (AI summary service box; Key Flow A "QR consult"), `Atlas LLD.dc.html` (AI RAG Service component, AI Providers external component, QR-consult sequence diagram steps 9–12), `Atlas_Project_Requirements.md` (§30 MVP scope, deferred items).

---

## 1. Overview and scope

Atlas ships four AI capabilities in the MVP, all served by one **AI summary service**: a stateless service that sits over the records API and generates text strictly from the requester's *authorized slice* of the record (see [07-consent-and-access-control.md](07-consent-and-access-control.md) for slice semantics). It never writes to the medical record, never trains on tenant data, and every generation is logged for review.

| # | Capability | Audience | Trigger | Prototype evidence |
|---|-----------|----------|---------|--------------------|
| 1 | Patient plain-language summary | Patient | New records published to the patient | Patient home "AI SUMMARY" card |
| 2 | Clinical brief | Doctor | QR scan (or break-glass emergency access) resolving to a patient 360° view | Doctor overview "AI CLINICAL SUMMARY" card with generating state |
| 3 | Ask-the-record Q&A | Doctor | Suggested-question chip tap or free-form question on the 360° view | Three chips + answer card under the clinical brief |
| 4 | Report explanation | Patient | Per-report request from the report detail screen | Architecture §10.1 (not rendered in the prototype; see §5) |

**Explicitly deferred (post-MVP)** — from PRD §30 ("Advanced AI/diagnostic capabilities", "Automated clinical decision support") and Architecture §10.1: drug-interaction detection, coding suggestions, and clinical decision support. The pipeline in §7 is designed to host these later without rework, but no MVP surface may present AI output as a diagnostic recommendation. (Note: the deterministic allergy-conflict warning in the doctor prescribing flow is a rules check on structured allergy data, not an AI feature — it is specified in [03-doctor-app.md](03-doctor-app.md) and is not routed through this service.)

Non-negotiable principles (Architecture §9 row "AI summary", §10):

- Generated **only from the requester's authorized slice** — role + consent categories + tenant, resolved per request.
- **Citations** to source records on every factual claim.
- **Fixed disclaimers** on every surface: patient-facing "not medical advice", clinician-facing "verify before clinical decisions".
- **Per-tenant configuration** (enable/disable, provider routing, precompute).
- **No cross-tenant training**; prompts and outputs logged for review.
- **Human-in-the-loop by design**: AI output is never written to the record and triggers no clinical action.

### Acceptance criteria

- The AI summary service exposes exactly the four capabilities above; no diagnostic or decision-support output path exists in the MVP.
- Disabling AI for a tenant removes all four surfaces for that tenant's users without degrading any non-AI screen.
- No AI-generated text is ever persisted as part of the clinical record.

---

## 2. Patient plain-language summary

A standing card on the patient home screen that reads the patient's recent record activity — new reports, diagnoses, and medication changes — back to them in plain language (Architecture §10.1).

### 2.1 Placement and visual treatment

The card sits on the patient home screen directly below the "My Atlas QR" card and above "Today's medication" (see [02-patient-app.md](02-patient-app.md) for the full home layout). Exact treatment from the prototype:

| Property | Value |
|----------|-------|
| Card background | `linear-gradient(135deg, oklch(0.96 0.02 235), oklch(0.94 0.03 260))` |
| Border / radius | `1px solid oklch(0.86 0.05 245)` / `16px` |
| Header icon | Rotated rounded square ("diamond"), fill `oklch(0.5 0.14 260)`, 15×15 |
| Header label | `AI SUMMARY` — 11px, weight 600, letter-spacing 0.06em, color `oklch(0.42 0.12 260)` |
| Freshness stamp | Right-aligned, 10.5px, 45% ink — e.g. `Updated today, 9:15` |
| Body text | 13.5px, line-height 1.55, ink `#22303c` |
| Provenance + disclaimer line | 10.5px, 45% ink, 8px below body |

Demo copy (Priya Sharma persona), fixed by the prototype and to be used as the canonical example in tests:

> Your new lipid panel looks mostly good — cholesterol is in range, but triglycerides are slightly high. Your diabetes remains well controlled (HbA1c 6.8%). Keep taking both medications; Dr. Menon may discuss diet at your 12 Sep follow-up.

Footer line, exact copy pattern:

```
Generated from 7 records · not medical advice — ask your doctor
```

### 2.2 Content rules

The summary must:

1. Lead with **what is new** — the most recently published report, diagnosis, or medication change ("Your new lipid panel…").
2. Interpret results in plain language relative to reference ranges ("cholesterol is in range", "triglycerides are slightly high") without raw jargon; specific values may appear where they aid understanding (HbA1c 6.8%).
3. State the **control status of ongoing conditions** ("Your diabetes remains well controlled").
4. Reinforce the **current medication plan** ("Keep taking both medications") — restating the record, never proposing changes.
5. Reference **upcoming follow-ups and the treating clinician by name** where they exist in the record ("Dr. Menon may discuss diet at your 12 Sep follow-up").
6. Contain **no advice that is not already present in the record** — it rephrases; it does not recommend. Anything resembling a new clinical instruction must fail the verify stage (§7, step 5).
7. Be generated in a calm, non-alarming register; abnormal flags are described factually ("slightly high"), never with urgency language. Proposed: a severity escalation ("contact your doctor about this result") requires an explicit tenant-configured rule, not model judgment.

Proposed: the summary renders in the patient's app language (English/Hindi/Marathi are supported app-wide per the approved gap recommendations); the generation request carries the target language, and the disclaimer line is a fixed translated string, never model-generated.

### 2.3 Update cadence

- The summary is **regenerated when new records publish** to the patient (Architecture §10.1). The triggering events are `report.finalized` / `record.created` on the event bus (Architecture §12 lists "AI pre-computation (where enabled)" as an async consumer). The prototype demonstrates the cadence: the lipid panel notification lands at 9:12 and the summary is stamped "Updated today, 9:15".
- Consent changes do not trigger regeneration here (the patient always sees their own full record), but a **model version upgrade invalidates the cache** and the next home view regenerates (§9).
- The freshness stamp always reflects the last successful generation, formatted `Updated {relative-day}, {H:MM}`.

### 2.4 "Generated from N records"

`N` is the count of **distinct source records in the retrieved grounding set** for this generation (§7, step 2) — not the total record count. It is computed by the pipeline, returned alongside the text, and rendered verbatim. Tapping the card is not wired in the prototype; Proposed: tapping opens a citation view listing the N source records, each deep-linking to its record detail.

### 2.5 States

| State | Behavior |
|-------|----------|
| Fresh | Card shows text + stamp + footer as above. |
| Stale-while-regenerating | Proposed: previous summary remains visible with its old stamp while a new one generates in the background; no skeleton on the patient home. |
| No records yet | Proposed: card hidden entirely for a patient with no published records (nothing to summarize). |
| Generation failure | Proposed: card hidden or shows last good summary with its stamp; the home screen never blocks on AI. |
| Tenant AI disabled | Card absent. |

### Acceptance criteria

- The card renders the exact header label, footer copy pattern, and disclaimer wording above; the disclaimer string is fixed and cannot be altered by model output.
- Publishing a new report to a patient results in a regenerated summary whose stamp is newer than the report's publish time.
- Every sentence in the summary is traceable to a record in the grounding set; a summary containing an unverifiable claim is not shown (§7 verify).
- `N` in "Generated from N records" equals the size of the retrieved grounding set for that generation.

---

## 3. Scan-time clinical brief

A synthesized clinical picture generated for the doctor at the moment a QR scan (or emergency break-glass) resolves into the patient 360° view. It is the first card on the Overview tab, above the structured Conditions / Current medications / Latest results cards (see [03-doctor-app.md](03-doctor-app.md)).

### 3.1 Trigger and flow position

From Canvas Key Flow A and the LLD QR-consult sequence diagram, the brief is the penultimate step of the scan flow:

```
Doctor scans QR → Gateway: tenant + rate + nonce → QR token → patient id
  → Policy: role + consent + categories → Compose 360° (authorized slice)
  → AI brief → Audit + patient alert
```

LLD sequence steps 9–12: after the policy PERMIT (step 8), the API Gateway issues `GET /patients/{id}/summary` to the Records Svc → Records Svc calls AI RAG `generateBrief(authorizedSlice)` → AI RAG returns `brief + citations (claims verified)` → Records Svc replies to the Doctor App `200 · 360° view + AI brief`. The Records service composes the consent-trimmed cross-tenant slice; the AI service generates **from that slice only** (Architecture §16, Flow A). QR mechanics are in [08-qr-subsystem.md](08-qr-subsystem.md).

The 360° view renders immediately after authorization; the brief loads asynchronously inside its card. The structured cards never wait on the AI.

### 3.2 Generating state

While the brief is being generated the card shows, exactly as prototyped:

- Header row: diamond icon + `AI CLINICAL SUMMARY` (11px, weight 600, letter-spacing 0.06em, `oklch(0.42 0.12 260)`), with right-aligned text `Generating…` in the same color.
- Body: three skeleton bars — height 9px, radius 5px, background `oklch(0.88 0.03 250)`, widths 95% / 85% / 60%.

The prototype resolves the state after ~1.6 s; the product SLO is **p95 < 2.5 s** (§9). The card background/border match the patient AI card (`linear-gradient(135deg, oklch(0.96 0.02 235), oklch(0.94 0.03 260))`, border `oklch(0.86 0.05 245)`, radius 14px).

### 3.3 Content structure

The brief is a single dense paragraph in clinical register, ordered exactly as follows (each element grounded in the demo copy):

| Order | Element | Demo instance |
|-------|---------|---------------|
| 1 | Age / sex | `34 F` |
| 2 | Conditions with control status | `T2DM (2022), well controlled — HbA1c 6.8% (Jun)` |
| 3 | New results with abnormal flags (abnormal values **bold**) | `New lipid panel today: **TG 168 ↑**, LDL 104, otherwise in range` |
| 4 | Current medications with doses and start context | `On Metformin 500 BD + Atorvastatin 10 OD since Aug` |
| 5 | Surgical history | `Appendectomy Nov 2025, uneventful` |
| 6 | Allergy (always **bold**) | `**Penicillin allergy.**` |
| 7 | Suggested focus for this consult | `Suggested focus: hypertriglyceridemia — diet review vs. dose adjustment` |

Canonical demo copy (13px, line-height 1.55):

> 34 F, T2DM (2022), well controlled — HbA1c 6.8% (Jun). New lipid panel today: **TG 168 ↑**, LDL 104, otherwise in range. On Metformin 500 BD + Atorvastatin 10 OD since Aug. Appendectomy Nov 2025, uneventful. **Penicillin allergy.** Suggested focus: hypertriglyceridemia — diet review vs. dose adjustment.

Content rules:

- Abnormal result values and the allergy statement are always emphasized (bold) — they are the two elements a clinician must not miss.
- "Suggested focus" frames **what to look at, as alternatives to weigh** ("diet review vs. dose adjustment") — it names the clinical question the new data raises; it never asserts a diagnosis or instructs a treatment (that would be deferred CDS, §1).
- Elements absent from the slice are omitted, not filled in — e.g. a consent grant excluding the mental-health category (off by default in the consent flow) means the brief simply carries nothing from that category.
- Values, dates, and drug names must appear verbatim in the grounding set (§7 verify).

Footer line, exact copy pattern (10.5px, 45% ink):

```
Generated from 7 records at scan · verify against source data before clinical decisions
```

`7` follows the same `N` rule as §2.4. The "verify against source data" clause is the fixed clinician-facing disclaimer and appears on every brief.

### 3.4 Citations

The brief is returned with **citations into the source records** (Architecture §10.1, LLD step 11 "brief + citations"). Proposed rendering (the prototype does not show citation affordances): each claim carries a reference to its source record id; tapping a cited value navigates to that record in the Reports/Timeline tabs so "verify against source data" is one tap, not a search.

### 3.5 Emergency (break-glass) variant

Proceeding through break-glass lands the doctor on the same 360° Overview with the red `EMERGENCY ACCESS · unconscious patient` banner, and the brief generates identically (the prototype's emergency path sets the same generating state). The slice is the emergency slice defined in [07-consent-and-access-control.md](07-consent-and-access-control.md); the generation's audit entry records the emergency context (§10).

### 3.6 Session end and lifecycle

- "End consult" clears the in-session Q&A state (prototype resets the open question/answer) and returns the doctor home with the toast `Consult ended · 1 session recorded`.
- The brief is **not stored in the patient record**. It may be served from the summary cache while the cache key is valid (§9); a re-scan after a new result or a consent change regenerates it because the slice hash changed.
- Where the tenant enables it, briefs are **precomputed on `report.finalized`** so the scan-time render is a cache hit (§9).

### 3.7 Failure state

Proposed (not shown in the prototype): if generation fails or exceeds its timeout, the card collapses to a single line — `AI summary unavailable — showing structured record` with a Retry affordance — and the structured cards below carry the consult. Failure never blocks or delays the 360° view, and the failure is logged (§10).

### Acceptance criteria

- Scanning a QR renders the 360° view immediately with the brief card in its generating state; the brief resolves at p95 < 2.5 s.
- The brief follows the seven-element order above; abnormal values and allergies are bold; the fixed "verify against source data before clinical decisions" footer is always present.
- Every number, date, and drug name in the brief appears in the retrieved slice; a brief that fails claim verification is refused, triggering the failure state, not shown.
- A brief generated under break-glass is audited as emergency access; a re-scan after consent revocation of a category produces a brief with no content from that category.

---

## 4. Ask-the-record Q&A

Free-form questions answered **strictly from the record** (Architecture §10.1 — examples given there: "last eye exam?", "HbA1c trend?"). The prototype demonstrates it via three suggested-question chips under the clinical brief; the architecture scopes the capability to free-form questions, so chips are the entry point, not the boundary.

**Audience decision (settling the deferral in [00-product-overview.md](00-product-overview.md) §4): ask-the-record is doctor-only in the MVP.** It is not offered on the patient summary surface. Rationale: the answer register is clinical (dense, dated, abbreviation-heavy — §4.3) and free-form patient Q&A over one's own record needs its own safety design (lay phrasing, escalation rules for alarming questions, "ask your doctor" boundaries) that the MVP does not carry. A patient-phrased variant is **Proposed post-MVP**; [12-api-and-interoperability.md](12-api-and-interoperability.md) §3.7 scopes `POST /v1/patients/{id}/ai/ask` to doctors accordingly. Patients' MVP AI surfaces remain the plain-language summary (§2) and per-report explanation (§5).

### 4.1 Suggested chips

Rendered in a wrapping row directly under the brief footer. Chip treatment: pill (999px radius), 11.5px weight 600, background `rgba(255,255,255,0.75)`, border `1px solid oklch(0.86 0.05 245)`, text `oklch(0.42 0.12 260)`, padding 5px 11px.

Proposed: chip questions are generated per patient by the same pipeline (they are questions the record can answer, biased toward gaps and trends relevant to the suggested focus); the three demo chips are fixed for the Priya persona.

### 4.2 Answer rendering

Tapping a chip renders one answer card inside the AI card: background `rgba(255,255,255,0.8)`, radius 10px, padding 10px 12px; the question in 11px weight 700 `oklch(0.42 0.12 260)`, the answer in 12.5px line-height 1.5. Tapping another chip replaces the visible exchange (the prototype holds one question/answer at a time). Ending the consult clears it. Proposed: answers carry the same citation affordance as the brief (§3.4).

### 4.3 The three demo exchanges (canonical examples)

These are fixed in the prototype (`aiMap`) and define the expected answer style — dense, dated, and honest about absence:

| Question | Answer (exact) | Pattern it demonstrates |
|----------|----------------|-------------------------|
| `Last eye exam?` | `No ophthalmology records in Atlas. Retinopathy screening was advised at diagnosis (Mar 2022) — no result on file. Consider referral.` | **Honesty about missing data**: states plainly that no data is on file, cites what the record *does* say (screening advised, when), and surfaces the natural next step. Never fabricates or infers an exam that is not recorded. |
| `HbA1c trend?` | `7.4% (Dec 2024) → 7.0% (Mar 2025) → 6.8% (Jun 2026). Improving; under 7% target for 14 months.` | **Trend synthesis**: every value/date pair is a real result in the slice; the interpretation ("Improving; under 7% target") is arithmetic over cited values, not clinical judgment. |
| `Med adherence?` | `Metformin refills on schedule for 12 months. Atorvastatin started 12 Aug — first refill due 11 Sep.` | **Derived-from-record inference**: adherence is inferred only from refill records present in the slice, with the basis stated. |

### 4.4 Grounding and refusal rules

- Answers are generated through the full pipeline (§7): the question is embedded, the slice is retrieved fresh, and claims are verified. A question whose answer is not supported by the slice must produce the "no data on file" pattern — never a guess and never knowledge from outside the record.
- The "Consider referral" construction is the ceiling of proactivity: naming the standard next step when the record explicitly shows a recommended action was never completed. Treatment or diagnosis suggestions beyond that are refused (deferred CDS, §1).
- Questions are user input crossing into the prompt: they are injected as data alongside the record content, inside the prompt-injection boundary (§7, step 3). A question that attempts instruction override is answered from the record or refused; it cannot alter disclaimers, scope, or the slice.
- Q&A shares the brief's authorized slice; consent revocation mid-session applies to the next question because retrieval re-runs per request (§7).

### Acceptance criteria

- The three demo exchanges reproduce exactly as tabled for the Priya persona.
- A question about data absent from the slice yields an explicit "no data on file"-style answer citing what the record does contain; it never fabricates.
- A prompt-injection attempt via the question text (or via document content retrieved into context) does not change disclaimers, reveal out-of-slice data, or alter behavior; these cases are covered in the red-team suite (§10).
- Answers to trend questions contain only values and dates present in the slice.

---

## 5. Report explanation

Per-report plain-language explanation of values and reference ranges for patients (Architecture §10.1). The prototype's report detail screen (Lipid panel: analyte table, provenance card, HbA1c trend, Download PDF / Share) does not render this block, so the UI is Proposed; the capability itself is committed architecture.

Proposed specification:

- **Placement**: an "Explain this report" card on the patient report detail screen, between the results table and the provenance card, styled as the §2 AI card family.
- **Content**: for each analyte — what it measures, where the value sits relative to the reference range, and what the flag means, in plain language; overall one-paragraph takeaway consistent in tone with §2.2 (calm, factual, no new advice). Reference ranges come from the report data itself; the model never supplies ranges from general knowledge (verify stage drops any range not present in the source).
- **Generation**: on-demand (first open), then cached under the standard key (§9) — a report is immutable per version, so the cache lives until model upgrade or report amendment (amendments re-embed and invalidate, §9).
- **Disclaimer**: same fixed patient disclaimer as §2 — `not medical advice — ask your doctor` — plus the `Generated from N records` provenance where N is normally 1 (the report), more if context records (e.g. the prior comparable panel) were retrieved.
- **States**: generating skeleton (as §3.2), failure collapses the card silently — the report itself is the primary content.

### Acceptance criteria

- Explanations mention only analytes, values, and ranges present in the report (plus retrieved comparison values); no externally-sourced reference ranges.
- An amended report version yields a fresh explanation; the superseded version's cached explanation is not shown.
- The card never delays or blocks rendering of the report values.

---

## 6. Deferred: diagnostic clinical decision support

Restating the boundary because every surface above must respect it. Deferred per PRD §30 and Architecture §10.1/§10.5:

- Drug-interaction detection (AI-driven; the deterministic allergy-conflict rule check remains in scope in [03-doctor-app.md](03-doctor-app.md)).
- Coding suggestions (ICD/medication coding assistance).
- Clinical decision support / diagnostic capabilities of any kind.

The RAG pipeline is explicitly designed to host these later "without rework" (Architecture §10.1) — meaning: new use case = new prompt template + model route + golden set, on the same authorize→verify skeleton. Until then, generation-side guardrails (§7 verify, §10 evaluation) must reject outputs that cross from *describing the record* into *recommending action*, with the two sanctioned exceptions already grounded in the prototype: "suggested focus" framing (§3.3) and "consider referral" on explicitly-recorded incomplete recommendations (§4.4).

---

## 7. RAG pipeline

Every generation — all four capabilities — runs this six-stage pipeline (Architecture §10.2). Grounding is computed **per request**; there is no pre-built cross-record corpus.

```
authorize → retrieve → assemble → generate → verify → respond
```

| Stage | Behavior |
|-------|----------|
| 1. Authorize | The policy engine resolves the requester's authorized slice: role + consent categories + tenant. Everything downstream sees only this slice. **Consent revocation takes effect instantly because retrieval is authorization-filtered at query time** — there is no index or cache consulted before the slice is resolved (caches are keyed by slice hash, so a changed slice is a miss, §9). |
| 2. Retrieve | Hybrid retrieval over the slice: (a) **structured pulls** from the FHIR-aligned records API — active problems, medications, and latest labs are *always* included; (b) **semantic search** over embedded record chunks for the long tail — old notes, discharge summaries, document text extracted at ingest. Output: the grounding set, each item carrying its record ID (this set's size is the `N` of "Generated from N records"). |
| 3. Assemble | Prompt built from normalized clinical facts with record IDs attached. Documents, notes, and user questions are injected **as quoted data, never as instructions** — this is the prompt-injection boundary. System instructions, disclaimers, and output contracts live outside the data channel. |
| 4. Generate | Call through the AI provider gateway (§8) with **per-use-case model routing**: small/fast model for patient summaries; stronger model for clinical briefs and Q&A. |
| 5. Verify | Post-processing checks **every factual claim against the retrieved facts — numbers, dates, and drug names must appear in sources**. Unsupported claims are dropped, or the whole response is refused when dropping would misrepresent it. Output filters catch PHI leakage outside the authorized slice (a name or value not in the slice never leaves the pipeline). |
| 6. Respond | Answer returned with inline citations back to source records and the fixed disclaimers ("not medical advice" / "verify before clinical decisions"). The prompt, the retrieved set, and the output are logged to audit (§10). |

Interfaces (from the LLD): the AI RAG Service provides `ISummary` — consumed through the API Gateway in the component diagram; in the QR-consult sequence the Records service invokes it as `generateBrief(authorizedSlice)`. Its drawn dependencies are the vector store (k-NN edge, §9) and the external AI Providers via the gateway's `complete() · embed()` calls (§8); retrieval over the records API and the stage-1 policy resolution follow Architecture §10.2. It owns no clinical data — Architecture §9 classifies it "stateless over records API"; its only stores are the disposable vector index and summary cache (§9). API surface details are in [12-api-and-interoperability.md](12-api-and-interoperability.md); service topology in [10-architecture.md](10-architecture.md).

Failure modes:

| Failure | Handling |
|---------|----------|
| Authorization resolves an empty slice | No generation; surface behaves as "no data" (§2.5, §3.7). |
| Retrieval degraded (vector store down) | Proposed: structured pulls alone can serve the brief and summary (they always include problems/meds/latest labs); Q&A over the long tail degrades to "no data on file" honesty pattern or unavailable state. |
| Provider failure | Gateway failover chain (§8); exhausted chain → surface failure state. |
| Verify rejects the output | Response refused; surface failure state; rejection logged with the unsupported claims for evaluation (§10). |

### Acceptance criteria

- Revoking a consent category and immediately re-requesting any AI output yields a generation containing nothing from that category — with no cache flush step required.
- Active problems, current medications, and latest labs appear in the grounding set of every brief regardless of semantic retrieval results.
- An output containing a number, date, or drug name absent from the grounding set is never returned to a client.
- Instructions embedded in an uploaded document's text do not alter generation behavior (verified by the §10 red-team suite).

---

## 8. Pluggable provider gateway

All model calls go through an internal **AI provider gateway** — one interface, N adapters. **No service talks to a vendor SDK directly** (Architecture §10.3).

### 8.1 Interface

```
complete(request) -> completion     // text generation
embed(request)    -> vector(s)      // embedding generation (ingest + query time)
```

Both calls carry: use case (patient_summary | clinical_brief | ask_record | report_explanation | embedding), tenant routing context, and pinned model identifier. Adapters translate to vendor APIs; callers never see vendor-specific types.

### 8.2 Adapters

| Adapter | Typical use | Notes |
|---------|-------------|-------|
| Azure OpenAI | Tenants on Azure / existing Microsoft BAA | Regional deployments; no-training + zero-retention flags |
| AWS Bedrock (Claude) | Default for AWS-hosted cells | In-region endpoints; model version pinning |
| GCP Vertex | GCP-resident tenants | Same contract via adapter |
| Self-hosted (vLLM) | Government / dedicated cells with no-egress rules | Open-weight models inside the tenant cell |

### 8.3 Per-tenant routing policy

Provider, model, and region are **tenant configuration**, chosen by residency, compliance posture, and cost tier (tenancy tiers are defined in [10-architecture.md](10-architecture.md)). **Switching providers is a config change, not a code change.** Routing is per use case within a tenant — e.g. a small/fast model route for patient summaries and a stronger route for briefs (§7 stage 4) can point at different models of the same provider or, Proposed, different providers.

### 8.4 Contractual requirements

- **BAA/DPA per provider** before any tenant routes to it.
- **Zero data retention** and **no training on tenant data** are hard requirements — deal-breakers, not preferences.
- Requests carry **no tenant identifiers beyond what the contract covers**.

### 8.5 Resilience, pinning, metering

- **Failover chains per use case**: primary → fallback provider/model, with timeouts and circuit breakers at the gateway. A tripped breaker fails over without caller involvement; an exhausted chain surfaces the §7 provider-failure mode.
- **Model version pinning with staged upgrades**: every tenant runs a pinned model version; upgrades roll out gated by the evaluation harness (§10) and invalidate summary caches (§9). No silent vendor-side model drift.
- **Per-tenant token metering** on every call, feeding the control plane's billing (see [10-architecture.md](10-architecture.md)).

### Acceptance criteria

- Grepping the codebase finds vendor SDK imports only inside gateway adapters.
- Re-pointing a tenant from one adapter to another requires only configuration and produces no code deployment.
- A provider outage on the primary route serves briefs via the failover chain within SLO or surfaces the failure state — never a hang.
- Every generation and embedding call produces a metering record with tenant, use case, model, and token counts.

---

## 9. Embeddings and caching

### 9.1 Vector store

- Embeddings are generated **at ingest** (and re-embedded on amendment) into a **per-tenant, encrypted vector store** — pgvector or OpenSearch k-NN (Architecture §10.4, §19) — **scoped by patient**.
- The store is **disposable and rebuilt from source of truth**: it is an index, never a system of record; a rebuild replays ingest-time embedding over the records.
- **Vectors are treated as PHI**: the same encryption, residency, and deletion obligations as the records they derive from. Deleting or amending a record deletes/replaces its vectors; tenant residency pins the vector store to the tenant's region; tenant offboarding destroys the partition (see [13-non-functional.md](13-non-functional.md)).

### 9.2 Summary cache

```
cache key = { patient, authorized-slice hash, model version }
```

- A **new record**, a **changed consent** (slice hash changes), or a **model upgrade** invalidates the entry by construction — no explicit invalidation bus is needed for correctness, though Proposed: entries are also evicted eagerly on `record.created` / `consent.granted` / `consent.revoked` events to reclaim space.
- Because the requester's slice hash is part of the key, two doctors with different consent grants never share a cached brief, and a patient's own summary never serves a doctor's request.

### 9.3 Latency SLO and precompute

- **Scan-time brief SLO: p95 < 2.5 s** from authorized request to rendered brief.
- Where the tenant enables it, briefs are **precomputed on `report.finalized`** (the "AI pre-computation" async consumer, Architecture §12) so the common scan-after-new-result path is a cache hit. Precompute uses the same pipeline and the patient's-record slice hash space; a scan whose slice differs from the precomputed one regenerates.

### Acceptance criteria

- Dropping the vector store and rebuilding from records yields equivalent retrieval results; no data is lost.
- A record deletion request removes its chunks from the vector store within the same obligation window as the record itself.
- After a consent change, the next brief request is a cache miss and reflects the new slice.
- With precompute enabled, a scan immediately following `report.finalized` renders the brief as a cache hit well inside the SLO.

---

## 10. Safety and evaluation

### 10.1 Human-in-the-loop by design

- AI output is **never written to the record** and **triggers no clinical action** — it is read-only decoration over the authorized record view. The clinician acts on source data ("verify against source data before clinical decisions"); the patient is pointed to their doctor ("not medical advice — ask your doctor").
- The MVP **explicitly defers decision support** (§6); guardrails enforce the descriptive/prescriptive boundary at the verify stage.
- Disclaimers are fixed strings composed by the response stage, outside model output — a generation cannot omit, reword, or contradict them.

### 10.2 Evaluation harness

- **Clinician-reviewed golden sets per use case** — patient summary, clinical brief, ask-the-record, report explanation each carry a set of (record slice → expected-quality output) cases reviewed by clinicians.
- **Regression runs gate model and prompt changes**: no prompt edit, model version bump, or provider switch ships without passing the golden sets for every affected use case (this is the gate behind §8.5 staged upgrades).
- **Red-teaming for prompt injection via uploaded documents**: adversarial documents (instructions embedded in PDFs/notes, e.g. "ignore previous instructions", exfiltration attempts, disclaimer-stripping) are part of the standing suite; the §7 stage-3 data/instruction boundary and stage-5 filters must hold. Question-channel injection (§4.4) is in the same suite.
- Proposed: verify-stage rejections in production (§7) are sampled into the evaluation corpus so real failure modes grow the golden sets.

### 10.3 Generation audit trail

Every generation is logged with:

```
{ requester, authorized-slice hash, model (+ version), prompt, output }
```

- Logged to the audit service — append-only, hash-chained, WORM-retained (see [10-architecture.md](10-architecture.md) §Audit and [13-non-functional.md](13-non-functional.md)).
- **Reviewable per tenant, exportable for compliance** — a tenant can answer "what did the AI show this doctor at that scan, generated from what, by which model" for any past generation.
- The retrieved set is logged with the prompt (§7 stage 6), so any historical claim can be re-verified against exactly what grounded it.

### Acceptance criteria

- No code path persists AI output into any clinical store; record writes originate only from the flows in [03-doctor-app.md](03-doctor-app.md) and [11-integrations.md](11-integrations.md).
- A prompt or model change that regresses any golden set is blocked from release.
- Every entry in the red-team document suite generates either a compliant answer or a refusal — never leaked out-of-slice data, altered disclaimers, or followed embedded instructions.
- For any historical generation, the audit trail reproduces requester, slice hash, model version, full prompt, and full output.
