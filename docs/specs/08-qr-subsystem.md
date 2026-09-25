# 08 — QR Subsystem

Defines the QR token design, the QR token service, the patient present and doctor scan experiences, the end-to-end scan flow, failure handling, fallbacks, and the threat model for Atlas's primary access mechanism.

Sources: `design/uploads/Atlas_Project_Requirements.md` (§10 QR-Based Patient Access, §12, §13, §15, §17, §28, §31, §33), `design/Atlas Architecture.dc.html` (§5 edge/API tier, §7 authorization, §8 QR access subsystem, §15 API surface, §16 flow A, §17 SLOs, §18 degradation), `design/Canvas.dc.html` (QR token service, Key Flow A, invariants, cache layer), `design/Atlas LLD.dc.html` (component diagram QR Token Service, sequence "QR consult", domain model), `design/Atlas App.dc.html` (Patient QR present screen, Doctor QR scan screen, doctor home, break-glass screen, access log, notifications).

---

## 1. Purpose and core principle

QR is one of Atlas's primary interaction mechanisms (PRD §10). Each patient has an Atlas QR identity, expressing the platform's core concept: one patient → one identity → one QR → complete medical history → controlled doctor access.

The subsystem is governed by the PRD's core design principle (§33), which the product UI paraphrases for each role:

> **The QR code identifies the patient; authentication and authorization determine what the doctor can see.**

The QR code is an **access mechanism, not an authentication mechanism**. Scanning a QR never grants access by itself: the scanner must already be an authenticated doctor, and the policy engine (see [07-consent-and-access-control.md](07-consent-and-access-control.md)) decides what, if anything, is visible. The prototype surfaces this principle to both roles:

- Patient QR screen footer: "This code identifies you — it contains no medical data. The doctor sees only what you've authorized."
- Doctor scan screen footer: "The QR identifies the patient — your authorization decides what you see."

### Acceptance criteria

- Possession of a QR image (photo, screenshot, print) grants zero data access to an unauthenticated party.
- Decoding the QR payload offline yields no patient name, ID, demographics, or medical data.
- Every scan outcome (grant or deny) is decided by the policy engine, never by the QR content.

---

## 2. QR token design

### 2.1 Token properties

| Property | Specification | Source |
|---|---|---|
| Content | Opaque token resolvable only by Atlas — no PHI, no identifiers | Architecture §8; PRD §10 |
| Lifetime (TTL) | 300 s (5 min); short-lived by design | LLD sequence msg 3 `token {ttl: 300s, rotating}`; LLD component `IQRToken … rotate (TTL 5 min)` |
| Rotation | Automatic; the patient app fetches a fresh token as the current one nears expiry. Prototype pill: "Secure token · rotates in 4:52" (a countdown; the mock is a static mid-life snapshot) | Architecture §8; App QR screen |
| Revocation | Tokens can be revoked before expiry (patient sign-out, device loss, admin action) | PRD §10; Architecture §8 |
| Session binding | Resolution binds the token to the scanning doctor's authenticated session | Architecture §8; LLD msg 5 `resolve(token) + sessionBind` |
| Anti-replay | Each resolve consumes a nonce; the gateway applies stricter anti-replay and nonce checks on QR endpoints | Architecture §5, §8; LLD msg 4 `POST /qr/resolve {token, nonce}` |
| Rate limiting | Per doctor and per tenant on resolution | Architecture §8 |
| Storage | Token state lives in a replicated Redis-class cache with strict TTL, tenant-namespaced keys, no durable PHI | Architecture §8, §13 data stores; Canvas cache layer |

The PRD (§10) requires the QR mechanism to support: expiration, rotation, revocation, session binding, rate limiting, anti-replay protection. All six are covered above.

### 2.2 Token format

Proposed (format not fixed by the sources; constraints are):

```text
QR payload (the only thing encoded in the QR image):
  atlas:v1:<token>

  <token> = 256-bit cryptographically random value, base64url-encoded
            (43 chars). Opaque: carries no structure, no patient reference,
            no tenant reference, no timestamp. All meaning lives server-side.
```

- The `atlas:v1:` prefix allows the scanner to reject non-Atlas QR codes instantly and versions the payload format. Proposed.
- The token is a random lookup key, not a signed claim (no JWT): nothing decodable client-side, and server-side revocation is immediate. Proposed rationale, consistent with "resolvable only by Atlas."
- QR error-correction level M, rendered inside the prototype's 250×250 white card (18 px padding, so the code itself is ≈214 px). Proposed.

### 2.3 Server-side token state

Proposed record shape (held in cache only, TTL-bound, no durable PHI — per Canvas: "Redis: sessions, QR token state, rate counters — keys namespaced by tenant · TTL-bound, no PHI"):

```text
key:   qr:token:{token_hash}
value: {
  atlas_patient_id,        # resolution target (an identifier, not PHI payload)
  issued_at, expires_at,   # 300 s window
  issuing_device_id,       # patient device that requested it
  status,                  # ACTIVE | REVOKED | CONSUMED
  resolve_count            # for replay detection thresholds
}
ttl:   expires_at + small grace for audit correlation (Proposed)
```

There is no durable `QRToken` entity in the domain model ([09-data-model.md](09-data-model.md)) — the LLD class model deliberately omits it; token state is ephemeral cache state, while the durable trace of QR usage is the audit log and the consent grant it produces.

### Acceptance criteria

- A token resolved more than 300 s after issue fails with an expired-token result.
- A revoked token fails resolution immediately (no cache-TTL lag beyond replication delay).
- Decoding any issued QR yields only the opaque payload; no field of it correlates to the patient without a server round-trip.
- Token state disappears from the cache at TTL; no PHI is ever written to the cache.

---

## 3. QR token service

### 3.1 Responsibilities and interface

The QR token service is a dedicated component (LLD component diagram: "QR Token Service — ⊙ IQRToken — issue · resolve · revoke · rotate (TTL 5 min)"; Canvas: "Issues opaque, short-lived rotating tokens · no PHI in QR · expiry, revocation, session binding, rate limits, anti-replay").

```text
IQRToken
  issue(tenantCtx, patient_session)  -> { token, ttl: 300, expires_at }
  resolve(token, nonce, doctor_session) -> { atlas_patient_id } | error
  revoke(token | patient_id scope)   -> void
  rotate                              (implicit: issue supersedes prior token)
```

External REST surface (PRD §28 lists `/qr`; LLD sequence fixes the two operations):

| Route | Caller | Behavior |
|---|---|---|
| `POST /qr/tokens` | Patient app (authenticated patient session) | Issues a fresh rotating token; returns token + TTL. Called on QR-screen open and again as the countdown nears zero. |
| `POST /qr/resolve` | Doctor app (authenticated doctor session) | Body `{token, nonce}`. Validates, session-binds, consumes nonce, returns `atlas_patient_id`. Never returns clinical data — composition happens afterwards via the records service. |
| `POST /qr/tokens/{id}/revoke` | Patient app / platform | Proposed explicit route for revocation (interface method exists; route shape per [12-api-and-interoperability.md](12-api-and-interoperability.md) §3.6). |

The API gateway sits in front and, for QR endpoints specifically, applies tenant resolution, per-tenant rate limits and quotas, and stricter anti-replay nonce checks before the service is reached (Architecture §5; LLD gateway box "anti-replay nonce (QR)").

### 3.2 Data owned

- Ephemeral token state and nonce set in the tenant-namespaced Redis-class cache. Nothing durable.
- Both issuing and resolution are audited (Architecture §8) — audit records live in the audit store, owned by the audit service, not this service.

### 3.3 Failure modes

| Failure | Behavior |
|---|---|
| Cache unavailable | Token issue/resolve degrade; patient-side QR presentation continues from client cache (Architecture §18: "QR presentation and the emergency card serve from cache if core services degrade; reads outlive writes"). Resolution requires the service; doctors fall back to search or break-glass (§8 below). |
| Replication lag on revoke | Proposed: revocations write-through to all replicas before acknowledging; worst-case exposure bounded by the 300 s TTL. |
| Clock skew | Proposed: expiry validated against server time only; client countdown is cosmetic. |

### Acceptance criteria

- `POST /qr/tokens` requires an authenticated patient session; `POST /qr/resolve` requires an authenticated doctor session (see [06-identity-auth.md](06-identity-auth.md)).
- `POST /qr/resolve` returns at most an identity reference, never clinical content.
- Every issue and every resolve attempt (success or failure) produces an audit event.
- QR-to-patient resolution meets p95 < 500 ms (Architecture §17 SLO; PRD §31 "near-instantaneous").

---

## 4. Patient present experience

### 4.1 Entry points

| Entry | Detail (from prototype) |
|---|---|
| Home "My Atlas QR" card | White card: 84×84 QR thumbnail in a bordered tile, title "My Atlas QR", mono Atlas ID `ATL-82X92K`, link-styled action "Present to doctor". Tapping anywhere opens the present screen. |
| Tab bar center button | Raised dark circular button (46 px, ink `#22303c`) with a white QR glyph, centered in the 5-slot tab bar. |
| Profile | Atlas ID shown in mono under the patient name (context, not an entry point). |

### 4.2 Present screen layout (screen: "Patient QR present")

Full-screen takeover, dark ink background `#22303c`, white text, status bar area treated as dark (the device frame switches to light-on-dark).

| Zone | Content | Exact spec from prototype |
|---|---|---|
| Top left | Dismiss | "‹ Close" — 14 px, white at 70 % opacity; returns to home. |
| Center stack (vertically centered, 18 px gaps) | Patient name | "Priya Sharma" — Instrument Serif, 26 px, white. |
| | QR code | 250×250 white rounded card (radius 20, padding 18, drop shadow `0 10px 40px rgba(0,0,0,0.35)`), dark-on-white QR. |
| | Atlas ID | `ATL-82X92K` — monospace (ui-monospace/Menlo), 14 px, white at 80 %. |
| | Secure-token indicator | Pill: `rgba(255,255,255,0.1)` background, radius 999, 7×16 px padding, 12.5 px text — green status dot (7 px, `oklch(0.8 0.15 150)`) + "Secure token · rotates in 4:52". |
| Bottom | Reassurance copy | "This code identifies you — it contains no medical data. The doctor sees only what you've authorized." — 12.5 px, white at 55 %, centered, max-width 280 px. |

### 4.3 Behavior

- On open, the app calls `POST /qr/tokens` and renders the returned token; the pill counts down from the token TTL (the prototype's "4:52" is a mid-life snapshot of the 5:00 window).
- At (or just before) 0:00 the app silently fetches a fresh token and re-renders the QR; the countdown resets. Proposed: a brief cross-fade on rotation so the patient understands the code changed; no interaction required.
- The green dot indicates a live, server-issued token. Proposed: the dot turns amber and the pill reads "Offline — code may need refresh" when the app cannot reach the server (see offline below).
- Proposed: while the present screen is open, the app raises screen brightness to maximum and restores it on close (standard scan-reliability behavior; not depicted in the mockup, flagged for review).
- Proposed: the OS screenshot of this screen is not blocked (tokens expire in ≤ 5 min, making screenshots self-defeating), but Android `FLAG_SECURE`/iOS capture detection may be revisited by security review.
- Family profiles: the home screen carries profile chips (Priya, Aarav, Kamala Sharma). Proposed: when a family member's profile is active, the present screen shows that member's name, Atlas ID, and QR — the guardian presents on their behalf (guardian links per [06-identity-auth.md](06-identity-auth.md)).

### 4.4 Offline presentation

The architecture requires the QR screen to be cached for offline display (Architecture §4 client layer: "The QR screen and emergency card are cached for offline display"; §18 graceful degradation). Specification:

- The app caches the most recently issued token and renders it offline.
- A cached token still expires server-side at its TTL; an offline patient may therefore present an expired code. Resolution failure then routes the doctor to fallbacks (§8). Proposed handling on the patient side: the countdown continues locally, and once elapsed the screen shows the code with the amber "may need refresh" state rather than blanking — the code is still useful as a visual pointer to the Atlas ID printed beneath it, and the doctor can type `ATL-82X92K` into search.
- The offline lock-screen emergency card (see [02-patient-app.md](02-patient-app.md)) is a separate, non-QR artifact and never embeds a token.

### Acceptance criteria

- Opening the present screen issues a fresh token; the rendered QR changes at every rotation.
- Name, mono Atlas ID, secure-token pill with live countdown, and the "contains no medical data" copy are all present exactly as specified.
- Closing and reopening the screen never reuses an expired token when online.
- The screen renders (from cache) with no network connectivity.

---

## 5. Doctor scan experience

### 5.1 Entry point

Doctor home primary action card (dark ink `#22303c`, white text, radius 16): icon of a stylized scan frame, title "Scan patient QR", subtitle "Start a consult — identity & consent checked automatically". Below it sits the search field ("Search by name, Atlas ID or phone" — the non-QR path, §8) with helper text "Results show identity only — records open after consent check", and a renewal hint card: "**Access expires** — Priya Sharma consent ends 12 Sep 2026. Re-scan her QR to renew." (re-scanning is also the renewal gesture for an expiring consult grant).

### 5.2 Scan screen layout (screen: "Doctor QR scan")

Full-screen camera takeover, near-black background `#101820`, white text.

| Zone | Content | Exact spec from prototype |
|---|---|---|
| Top left | Dismiss | "‹ Cancel" — 14 px, white at 70 %; returns to doctor home. |
| Center | Viewfinder | 240×240 region, subtle `rgba(255,255,255,0.04)` fill, four corner brackets (44×44, 3.5 px stroke, accent `oklch(0.72 0.1 235)`, 18 px corner radius). Live camera feed fills the screen behind it (the prototype's "Simulate scan" button is mock-only scaffolding standing in for code detection; Proposed: the product uses the camera and auto-detects, no shutter button). |
| Below viewfinder | Helper copy | "Align the patient's Atlas QR within the frame" — 14 px, white at 70 %, centered, max-width 250 px. |
| | Emergency fallback link | "Emergency access — no QR available" — 13 px semibold, warm red `oklch(0.75 0.12 25)` with underline; opens break-glass (§8.2, [07-consent-and-access-control.md](07-consent-and-access-control.md)). |
| Bottom | Principle copy | "The QR identifies the patient — your authorization decides what you see." — 12 px, white at 45 %, centered. |

### 5.3 Scan states

| State | UI | Trigger |
|---|---|---|
| Scanning (idle) | Viewfinder + helper copy | Screen open, no code detected |
| Resolving | Detected code shown bright inside the frame; status line "Identity resolved · checking consent…" in light accent `oklch(0.8 0.1 235)` | Code decoded; `POST /qr/resolve` then policy evaluation in flight (~1 s in the prototype) |
| Granted | Navigate to the patient 360° view (Overview tab), AI clinical summary in its "Generating…" skeleton state; toast: "Access granted · consult session logged in audit trail" | PERMIT from policy engine |
| Denied / failed | See §7 | Any failure path |

After grant, the 360° header shows the audited consult-session pill ("Session 28:44 · audited" — a countdown; Proposed: sessions are 30 minutes, extendable by activity) and the severe-allergy banner; "End consult" returns home with toast "Consult ended · 1 session recorded". Full 360° spec: [03-doctor-app.md](03-doctor-app.md); AI brief: [05-ai-features.md](05-ai-features.md).

### Acceptance criteria

- A doctor must be authenticated (org SSO + MFA) before the scan screen is reachable.
- Scan auto-detects without a shutter press (Proposed interaction); a valid Atlas QR moves to "Identity resolved · checking consent…" and then to grant or deny.
- Total scan-to-360° experience meets the SLOs: token resolution p95 < 500 ms, 360° load < 1 s.
- The emergency-access link and the principle copy are present on the scan screen.

---

## 6. End-to-end scan flow

Authoritative sequence (LLD "Sequence · QR consult", cross-checked against Architecture §16 Flow A and Canvas Key Flow A):

```text
 1. Patient App  → API Gateway   POST /qr/tokens
 2. API Gateway  → QR Token Svc  issueToken(tenantCtx)
 3. QR Token Svc → Patient App   token {ttl: 300s, rotating}          [reply]
    — patient presents; doctor scans —
 4. Doctor App   → API Gateway   POST /qr/resolve {token, nonce}
    (gateway: resolve doctor's tenant · rate limits · nonce check)
 5. API Gateway  → QR Token Svc  resolve(token) + sessionBind
 6. QR Token Svc → API Gateway   atlas_patient_id                     [reply]
 7. API Gateway  → Policy Engine evaluate(actor, patient, purpose)
 8. Policy Engine→ API Gateway   PERMIT + consultGrant(30d, categories) [reply]
 9. API Gateway  → Records Svc   GET /patients/{id}/summary
10. Records Svc  → AI RAG        generateBrief(authorizedSlice)
11. AI RAG       → Records Svc   brief + citations (claims verified)  [reply]
12. Records Svc  → Doctor App    200 · 360° view + AI brief           [reply]
13. API Gateway  → Audit         «async» audit: scan · grant · view
14. API Gateway  → Notification  «async» notify patient (PHI-free)
```

Key semantics:

- **Tenant handling.** The patient is a platform-level (global) identity; the doctor's tenant is resolved at the gateway from subdomain/token claim and stamped as signed tenant context. The scan is precisely the moment a tenant-scoped actor gains consented access to a global identity's cross-tenant record — Canvas invariant 6: "Cross-tenant reads exist ONLY via patient-consented composition, always audited."
- **Implicit consult grant.** A successful scan creates a consent grant with default duration — 30 days per the LLD (`consultGrant(30d, …)`) and the prototype's patient-side evidence (access log: "12 Aug 10:40 — QR scanned · access granted (30 days)"; notification "Access granted to Dr. R. Menon · 30 days · 12 Aug"). Default categories follow the consent model: general history, reports & labs, medications on; mental health and other sensitive categories off by default. Grant lifecycle, category set, and revocation: [07-consent-and-access-control.md](07-consent-and-access-control.md). Re-scanning while a grant is active renews it (doctor home: "Re-scan her QR to renew").
- **360° composition.** The records service composes the authorized 360° slice — cross-tenant, consent-trimmed — and only that slice feeds the AI clinical brief (Architecture §16 A.5). See [10-architecture.md](10-architecture.md) and [05-ai-features.md](05-ai-features.md).
- **Audit.** Three audit events minimum per successful scan — scan, grant, view — appended to the hash-chained audit store with `{actor, role, tenant, patient, action, resource, access_method: QR, result, session}` (Canvas audit service; PRD §13 lists "Patient accessed via QR" as a first-class event). Token issue (step 2) is also audited (Architecture §8).
- **Patient notification.** PHI-free push to the patient: the prototype shows "Dr. R. Menon viewed your records · via QR scan · 12 Aug". The patient-visible access log mirrors it ("12 Aug 10:41 — Dr. R. Menon · viewed via QR").

### Acceptance criteria

- One successful scan yields: exactly one resolved identity, one active consult grant (created or renewed), one composed 360° response, ≥ 3 audit events, and one patient notification.
- The consult grant is visible immediately in the patient's Access & consent screen with its 30-day expiry.
- Steps 13–14 are asynchronous and never block the doctor's 360° response.
- No response at any step contains data outside the policy engine's authorized slice.

---

## 7. Failure paths

The prototype depicts only the happy path; the PRD and architecture mandate the failure controls (expiry, revocation, anti-replay, rate limits). Handling below is therefore largely **Proposed** and flagged for design review; the states themselves are grounded.

| # | Failure | Detection | Doctor-side handling (Proposed copy) | Audit |
|---|---|---|---|---|
| F1 | Expired token | `expires_at` passed at resolve | Inline sheet on scan screen: "This code has expired. Ask the patient to open Atlas — the code refreshes automatically." Primary action "Scan again"; secondary "Search by ID instead". | `qr.scan_failed (expired)` |
| F2 | Revoked token | `status = REVOKED` | Same UX as F1 (indistinguishable to the doctor by design — no information about why). | `qr.scan_failed (revoked)` |
| F3 | Unknown / malformed token | No cache entry, or payload not `atlas:v1:` | Non-Atlas code: passive "Not an Atlas code" hint, keep scanning. Well-formed but unknown token: same generic failure as F1 — never confirm whether a token ever existed. Counts toward the scanner's rate limit. | `qr.scan_failed (unknown)` |
| F4 | Replayed nonce / token reuse anomaly | Nonce already consumed; abnormal `resolve_count` | Resolution refused with the generic failure; repeated anomalies flag the doctor's session for review. | `qr.scan_failed (replay)` + security alert |
| F5 | Rate limit exceeded | Per-doctor or per-tenant limiter at gateway | "Too many scan attempts — try again shortly." Scan disabled with visible cool-down. | `qr.scan_throttled` |
| F6 | Consent denied (policy DENY) | Policy engine at step 7 | Identity was resolved, so show the identity-only result (name, age/sex, Atlas ID — mirroring search's "Results show identity only — records open after consent check") with action "Request access", which starts the consent-request flow (Flow C, [07-consent-and-access-control.md](07-consent-and-access-control.md)): the patient receives a PHI-free notification, reviews requester, duration (24 h / 30 d / 90 d) and category toggles, and approves or denies. No clinical data is shown meanwhile. | `qr.scan_resolved` + policy evaluation logged with `result = denied` (07 §8.2) |
| F7 | Patient revokes mid-session | Grant status change; policy cache invalidated (seconds-scale) | Open 360° session ends at next request: "Access ended — the patient revoked consent." Return to home. | `access.revoked` |
| F8 | Core services degraded | Records/policy unreachable after resolve | Scan reports temporary unavailability; doctor may use break-glass if clinically urgent. Patient-side presentation keeps working from cache. | best-effort, reconciled |

Design rules across all failures:

- Failure responses are uniform and information-free: a failed resolve never distinguishes expired vs revoked vs never-existed to the caller (F1–F4 share one generic doctor-facing message), preventing token-space probing. Proposed.
- Every failure is audited with the attempting doctor, tenant, and reason code; failures never contain PHI (PRD §13, §17).
- F6 is the designed bridge from "QR identified the patient" to "patient controls what is seen" — deny is a normal outcome, not an error.

### Acceptance criteria

- Expired, revoked, and unknown tokens produce the same doctor-facing message and distinct audit reason codes.
- A DENY outcome shows identity only and offers the consent-request flow; no clinical field is rendered.
- Revocation takes effect against an in-flight consult session within seconds (policy cache invalidation window).
- Throttled scanners see an explicit cool-down; limits are enforced per doctor and per tenant.

---

## 8. Fallbacks when QR is unavailable

Three distinct non-QR paths exist; they must never be conflated.

| Path | When | Authorization model | Data reachable |
|---|---|---|---|
| **Patient search** | Patient present and cooperative but no scannable QR (dead phone, expired offline code) | Normal consent: results are identity-only; records open only after a consent check; otherwise a consent request is sent to the patient | Identity only → full authorized slice after patient approval |
| **Break-glass emergency access** | Patient cannot present a QR or give consent (e.g. unconscious) | Constrained emergency grant issued by the policy engine, mandatory reason, no patient approval | Emergency profile + critical history only |
| **Static printed QR** | Patient has no phone at all | Higher-friction resolution flow requiring an additional identity check | As per consent after the extra check |

### 8.1 Search

Doctor home search field: "Search by name, Atlas ID or phone" with helper "Results show identity only — records open after consent check" (PRD §15 additionally lists DOB and org-specific identifiers as search keys; results must not expose medical information to unauthorized users). The patient's Atlas ID (`ATL-82X92K`) is printed in mono on the home card, present screen, profile, and emergency card precisely so it can be read out or typed when scanning fails. Full search spec: [03-doctor-app.md](03-doctor-app.md).

### 8.2 Break-glass

Entered from the scan screen link "Emergency access — no QR available". The break-glass screen (dark red-tinted `#1c1214`) requires:

- A reason (chips: "Unconscious patient" / "Critical care" / "Other").
- A patient identifier — input placeholder: "Patient identifier (Atlas ID, ABHA or phone)". All three fallback identifiers resolve through the identity registry, not the QR token service.
- Explicit confirmation: button "Proceed — logged & reviewed"; footer "Compliance is notified immediately. Misuse ends access rights."

The policy engine issues a constrained emergency grant limited to the emergency profile plus critical history; compliance and the patient are alerted immediately, and the session is flagged in audit for post-hoc review (Architecture §16 Flow D). The resulting 360° view carries the red banner "EMERGENCY ACCESS · unconscious patient" with "compliance notified" right-aligned. Full spec: [07-consent-and-access-control.md](07-consent-and-access-control.md).

### 8.3 Static printed QR

For patients without phones, printed/static fallback QRs resolve to a higher-friction flow requiring an additional identity check (Architecture §8). Proposed detail: a static QR encodes a long-lived opaque pointer (not a 5-minute token); resolving it returns identity only and forces the doctor through an extra verification step (e.g. confirming DOB or phone with the patient) before any consent evaluation — compensating for the absence of rotation. Static QR issuance/reissue (and revocation on loss) is an admin/enrollment concern: [04-admin-app.md](04-admin-app.md).

### 8.4 Not a fallback: the offline emergency card

The patient's lock-screen emergency card ("Visible from the lock screen — no sign-in needed") shows name, blood group, allergies, conditions, medications, emergency contact, and the Atlas ID — it is locally rendered reference data, not a QR access path, though its printed Atlas ID can feed search or break-glass. Card opens are logged ("Anyone who opens this card is logged"). Spec: [02-patient-app.md](02-patient-app.md).

### Acceptance criteria

- Search never reveals more than identity before a consent check passes.
- Break-glass cannot proceed without both a reason and a patient identifier, and always triggers immediate compliance + patient alerts.
- Static-QR resolution demands an additional identity verification step before consent evaluation.

---

## 9. Threat model and rate limits

PRD §17 requires secure QR token design, rate limiting, and protection against replay attacks as core (not later) requirements.

| Threat | Vector | Mitigations |
|---|---|---|
| Replay | Captured `POST /qr/resolve` request re-sent | Single-use nonce consumed per resolve, enforced at gateway (stricter checks on QR endpoints) and service; 300 s TTL bounds the window; anomalous `resolve_count` flags review (Proposed threshold). |
| Screenshot / photo of the QR | Patient's code photographed and scanned later or elsewhere | 5-minute rotation makes captures stale almost immediately; resolution still requires an authenticated doctor and a consent PERMIT — a stale capture yields at most an identity resolution followed by DENY, fully audited. Proposed: resolving a token > N times or from multiple doctor sessions raises a security signal. |
| Relay ("scan proxying") | Attacker relays a live QR to a colluding authenticated doctor elsewhere | Session binding ties resolution to the resolving doctor's session; the resulting grant names that doctor, is patient-visible in the access log, and triggers the "viewed your records" notification — detection by transparency. Residual risk accepted and flagged in audit review. |
| Token guessing / enumeration | Brute-forcing the opaque token space | ≥ 256-bit random tokens (Proposed size); per-doctor and per-tenant rate limits on resolve; uniform information-free failures (§7); throttle-then-lock on repeated unknown-token attempts. |
| Stolen patient device | Attacker presents the victim's QR | The QR only identifies; it exposes nothing by itself. Patient app sign-in is protected by biometric/passkey ([06-identity-auth.md](06-identity-auth.md)); sign-out/device-loss revokes outstanding tokens. |
| Malicious QR shown to doctor | Doctor scans an attacker-crafted code | Payload-prefix validation; the token resolves only against Atlas's own cache; non-Atlas codes are ignored. The QR payload is never treated as a URL to open. |
| Insider over-scanning | Doctor scans patients without clinical need | Every scan audited with actor/tenant/purpose; per-doctor rate limits; patient notification on every access; break-glass reviewed by compliance. |

Rate limits (grounded requirement per Architecture §8 "rate-limited per doctor and per tenant"; numeric values Proposed for launch, tunable per tenant via the control plane):

```text
POST /qr/resolve   per doctor:  10/min, burst 5      (Proposed)
POST /qr/resolve   per tenant:  plan-based quota      (Proposed)
POST /qr/tokens    per patient device: 6/min          (Proposed; covers rotation + retries)
failed resolves    per doctor:  5 consecutive → 15-min cool-down + security event (Proposed)
```

### Acceptance criteria

- A replayed resolve request (same token + nonce) is rejected and audited.
- Uniform failure responses leak no distinction between expired, revoked, and nonexistent tokens.
- Rate limits are enforced per doctor and per tenant, with tenant-configurable quotas.
- Every scan, grant, deny, and break-glass event is patient-visible (access log) and compliance-visible (audit export).

---

## 10. Performance, availability, and degradation

| Requirement | Target | Source |
|---|---|---|
| QR-to-patient resolution | p95 < 500 ms | Architecture §17 SLOs; PRD §31 "near-instantaneous" |
| Patient 360° load after grant | < 1 s | Architecture §17 |
| Token state store | Replicated cache, strict TTL, tenant-namespaced | Architecture §8, §13 |
| Degraded mode | QR presentation and emergency card serve from client cache if core services degrade; reads outlive writes | Architecture §18 |
| Availability | High availability — doctors depend on Atlas during active patient care | PRD §31 |

### Acceptance criteria

- Resolution latency is monitored per tenant against the p95 < 500 ms SLO with dashboards and alerts (observability is tenant-tagged).
- Killing the cache tier degrades issuance/resolution but never crashes patient-side presentation.

---

## 11. ABDM scan-and-share alignment

Atlas is India-first and ABDM/ABHA-aligned (approved gap-analysis recommendation; onboarding copy "Works with ABHA (ABDM) · encrypted · you control access"; profile shows ABHA address `priya@abdm` with a "Linked" badge; ABHA is optional federation for patient identity per Architecture §6).

Alignment points for the QR subsystem:

- Atlas's pattern — patient-presented QR resolved to an identity, followed by consented data flow — is structurally congruent with ABDM's scan-and-share model (patient scans/presents at a facility; demographic sharing rides on ABHA consent). The Atlas QR remains an Atlas-opaque token; it does not embed the ABHA address (no identifiers in the QR).
- ABHA is a first-class fallback identifier: the break-glass identifier field explicitly accepts "Atlas ID, ABHA or phone", and the identity registry can resolve a linked ABHA address to the Atlas patient.
- Proposed (future, post-MVP, consistent with the architecture's "national platform federation" direction): an interop mode in which the patient present screen can additionally surface an ABDM-compliant scan-and-share payload for ABDM-participating facility desks, and in which Atlas can consume ABDM consent artefacts. This is explicitly not in MVP scope and requires its own compliance design.

### Acceptance criteria

- A linked ABHA address resolves to the correct Atlas patient in break-glass and search flows.
- The Atlas QR payload contains no ABHA address or other national identifier.

---

## 12. Cross-references

| Topic | Spec |
|---|---|
| Patient app screens hosting the QR entry points, emergency card | [02-patient-app.md](02-patient-app.md) |
| Doctor home, search, 360° view, consult session, add-to-record | [03-doctor-app.md](03-doctor-app.md) |
| AI clinical brief generated at scan | [05-ai-features.md](05-ai-features.md) |
| Patient/doctor authentication, sessions, device binding, ABHA federation | [06-identity-auth.md](06-identity-auth.md) |
| Policy engine, consent grants, categories, break-glass, revocation | [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| Domain entities (ConsentGrant, AuditEvent, Patient) | [09-data-model.md](09-data-model.md) |
| Gateway, tenant context, cache tier, degradation, SLO machinery | [10-architecture.md](10-architecture.md) |
| `/qr` API surface, versioning, idempotency | [12-api-and-interoperability.md](12-api-and-interoperability.md) |
| Performance, availability, audit retention requirements | [13-non-functional.md](13-non-functional.md) |
