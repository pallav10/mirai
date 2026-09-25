# Atlas Admin App

Specification of the organization-admin experience: the org dashboard (KPIs, integration health, management rows) and the integration error queue, plus the admin capabilities the PRD assigns to the organization role that the prototype does not yet depict.

Sources: `Atlas App.dc.html` (admin screens, lines ~691–745, and the logic script's admin state/actions), `Atlas_Project_Requirements.md` §3.3 (organization/admin role), §13 (audit trail), §21 (hospital integration configuration), §22 (patient identity resolution), §24 (report lifecycle & publication), §26 (integration error handling), §27 (provenance), `Atlas Architecture.dc.html` §6–7 (tenant directory, RBAC), §11 (integration pipeline and processing states), §16 (tenant onboarding), §17 (observability/DLQ alarms), `Atlas LLD.dc.html` (Admin Console component, IAdminAPI, integration activity flow), `Atlas Directions.dc.html` (1a Meridian visual direction).

---

## 1. Scope and role context

The Admin role represents the healthcare organization (the tenant). Per PRD §3.3 it manages doctors and staff, organization profiles, access policies, patient records created by the organization, audit logs, and organization-level configuration. Per the architecture document's RBAC model the role is `org-admin`, layered with ABAC attributes (tenant, facility) — an admin's authority never crosses tenant boundaries.

The prototype depicts the two admin surfaces that were prioritized: the **organization dashboard** and the **Legacy HIS integration error queue**. These are the implemented scope of this spec (§3–§5). Everything else PRD §3.3 grants the role is specified as Proposed in §7 so reviewers can distinguish prototype-grounded behavior from extrapolation.

In the LLD component diagram the Admin Console is its own client component ("«component» Admin Console — requires IAdminAPI — integrations · error queue"), separate from the Patient App and Doctor App. The prototype renders the admin screens inside the same mobile device frame purely as a prototyping convenience.

- **Proposed:** production form factor is a responsive web console (desktop-first) served to org-admin users, with the mobile-width layouts in this spec as the small-breakpoint rendering. Nothing in the layouts below requires more than a 390 px column, so the mobile rendering is shippable as-is; wider breakpoints may place the KPI tiles and management rows in a grid.
- All admin traffic goes through the API Gateway under `IAdminAPI` (see [12-api-and-interoperability.md](12-api-and-interoperability.md)); every request carries the signed tenant context.

### 1.1 Role switcher context

The prototype's top chrome shows three pills — `Patient`, `Doctor`, `Admin` — plus an iOS/Android device toggle. The active pill renders ink-filled (`#22303c` background, white text); inactive pills are transparent with ink text. Selecting `Admin` switches the frame to the admin dashboard (the admin sub-state defaults to the dashboard screen).

This switcher is **prototype chrome for reviewing all three roles in one canvas — it is not a product feature**. In the product:

- Patients and doctors get the mobile apps specified in [02-patient-app.md](02-patient-app.md) and [03-doctor-app.md](03-doctor-app.md).
- Admins sign in to the Admin Console with an org-admin identity. **Proposed:** admin authentication follows the doctor-side organization authentication model — organization/tenant IdP (SSO via OIDC/SAML per the architecture document's identity section) with MFA; see [06-identity-auth.md](06-identity-auth.md). No QR, OTP, or family-profile mechanics apply to this role.
- A user holding multiple roles (e.g., a doctor who is also an org-admin) authenticates once per surface; role/permission evaluation is the policy engine's job ([07-consent-and-access-control.md](07-consent-and-access-control.md)), not a client-side toggle.

### 1.2 Screen inventory and routes

| Screen | Prototype state | Entry | Exit |
|---|---|---|---|
| Org dashboard | `aScreen: 'dash'` | Default screen for the Admin role | Tap Legacy HIS card → error queue |
| Integration error queue | `aScreen: 'queue'` | Legacy HIS card on dashboard | `‹ Organization` back link → dashboard |

```text
/admin                          → Org dashboard
/admin/integrations/:id/events  → Integration error queue (prototype shows id = legacy-his)
```

**Proposed:** route names above are suggested; the prototype defines only the two-screen navigation relationship. Additional routes for §7 capabilities (staff, policies, audit, integration config) are listed there.

### Acceptance criteria

- An org-admin signing in lands on the org dashboard for their own tenant only; no cross-tenant data is reachable from any admin surface.
- The Admin Console is a distinct client of `IAdminAPI`; patient/doctor APIs are not reachable with an org-admin-only credential.
- The role switcher pills do not ship in any production client.

---

## 2. Shared visual chrome

Admin screens use the Meridian system exactly as the patient and doctor apps do (see [01-design-system.md](01-design-system.md)): warm paper screen background `#faf7f1`, ink `#22303c`, Instrument Serif for display headings, Instrument Sans for body, white cards with `1px solid rgba(34,48,60,0.1)` borders and 14–16 px radii. Secondary text uses ink at 50–55 % opacity. Confirmation feedback uses the standard dark toast (ink background, white 13 px text, bottom-anchored, ~2.8 s auto-dismiss).

Status colors used on the admin surfaces (verbatim from the prototype):

| Meaning | Color |
|---|---|
| Healthy indicator dot | `oklch(0.65 0.15 150)` (green) |
| Unhealthy indicator dot | `oklch(0.55 0.18 25)` (red) |
| Attention card border | `1.5px solid oklch(0.7 0.14 25)` |
| Error badge (FAILED / failed count) | bg `oklch(0.94 0.04 25)`, text `oklch(0.45 0.15 25)` |
| Warning badge (RETRYING) | bg `oklch(0.93 0.06 80)`, text `oklch(0.45 0.12 60)` |
| Neutral badge (DEAD_LETTER) | bg `rgba(34,48,60,0.1)`, text `rgba(34,48,60,0.65)` |

---

## 3. Org dashboard

Purpose: give the organization one glance at scale (staff, patients, integrations), integration health with a direct path to failures, and entry points to governance functions.

### 3.1 Layout (top to bottom)

**Header** — left-aligned text block, right-aligned org avatar:

- Eyebrow: `City General Hospital · Admin` — 13 px, ink at 55 %. Format: `{organization name} · Admin`.
- Title: `Organization` — Instrument Serif, 26 px.
- Org avatar: 40 × 40 px, 12 px radius, background `oklch(0.9 0.03 235)`, initials (`CG`) 13 px semibold in `oklch(0.42 0.12 235)`. Initials derive from the organization name.

**KPI tile row** — three equal-width white cards (16 px radius, 12 px padding, 10 px gap):

| Tile label (11 px, ink 50 %) | Value (Instrument Serif 22 px) | Data source |
|---|---|---|
| `Doctors` | `42` | Count of active practitioner accounts in this tenant (tenant directory) |
| `Patients` | `12,408` | Count of patients with records created by this organization |
| `Integrations` | `3` | Count of configured integrations for this tenant |

Values are live counts, formatted with thousands separators. The demo tenant City General Hospital shows 42 / 12,408 / 3. **Proposed:** tiles are read-only in v1 (the prototype attaches no tap action); tapping may later deep-link to the corresponding management screen.

**Integrations section** — section heading `Integrations` (Instrument Serif, 19 px), then one card per configured integration (white, 14 px radius, `13px 15px` padding, 8 px vertical gap). Each card: a 10 px status dot, a name line (14 px semibold), a meta line (12 px, ink 55 %), and — when unhealthy — a failed-count badge.

| Integration | Dot | Meta line copy | Badge | Tap |
|---|---|---|---|---|
| `Apollo LIS` | green | `FHIR · 1,204 events today · healthy` | — | none |
| `Radiology PACS` | green | `Webhooks · 96 events today · healthy` | — | none |
| `Legacy HIS` | red | `File-based · last sync 08:40` | `3 failed ›` | Opens error queue |

Meta line format: `{protocol} · {events today} events today · healthy` when healthy; `{protocol} · last sync {HH:MM}` when in the attention state. The attention-state card additionally gets the red `1.5px` border, and its failed badge shows the count of events currently sitting in that integration's error queue — RETRYING, FAILED, or DEAD_LETTER (Legacy HIS shows `3 failed ›`, matching the three queue entries in §4). The whole card is the tap target.

**Health derivation** — **Proposed** (the prototype fixes only the two visual states): an integration renders *healthy* (green dot, plain border) when it has zero events in FAILED/DEAD_LETTER and its last successful sync/event is within its configured sync interval; it renders *attention* (red dot, red border, failed badge) when either condition fails. This is the UI surfacing of the architecture requirement that DLQ depth alarms and integration lag are "surfaced to tenant admins" (see [13-non-functional.md](13-non-functional.md)). Threshold values come from per-integration error/retry configuration (PRD §21; [11-integrations.md](11-integrations.md)).

**Management rows** — three white rows (14 px radius, `14px 16px` padding), each a label (14 px semibold) with a right-aligned summary (12 px, ink 50 %) ending in `›`:

| Row | Right-side copy | Meaning |
|---|---|---|
| `Staff & roles` | `42 doctors · 8 admins ›` | Staff management entry; counts of active doctors and admins in the tenant |
| `Access policies` | `Publication rules ›` | Org access/publication policy entry (report publication rules, PRD §24) |
| `Audit log` | `Export · 30 days ›` | Audit review/export entry; export window is 30 days |

In the prototype these rows are static (no tap action). Their destination screens are specified as Proposed in §7; the rows themselves, their copy, and their live counts are in scope now.

### 3.2 States

- **Default:** as above, with live counts.
- **All healthy:** no integration card carries a badge or red border; the dashboard raises nothing. (Not shown in the prototype but implied by the health rules; **Proposed** as the trivial case.)
- **Proposed — loading:** KPI values and event counts render as blank card skeletons until loaded; never show stale counts as if current.
- **Proposed — no integrations configured:** Integrations section shows an empty-state card `No integrations configured` linking to integration setup (§7.3).

### 3.3 Interactions

- Tap Legacy HIS (any attention-state integration card) → integration error queue for that integration.
- Healthy integration cards, KPI tiles, and management rows have no tap behavior in the prototype (management-row destinations: §7).

### Acceptance criteria

- Dashboard header shows the signed-in admin's organization name; KPI tiles show live tenant-scoped counts (demo: 42 / 12,408 / 3).
- Every configured integration appears as a card with protocol and daily event count; health state matches the failed/DLQ and sync-recency rules.
- An integration with ≥1 event in FAILED or DEAD_LETTER renders the red dot, red border, and a `{n} failed ›` badge whose count matches the number of events in its error queue (RETRYING + FAILED + DEAD_LETTER; demo: 3), and tapping it opens that integration's error queue.
- A healthy integration card shows the green dot and `healthy` in its meta line and has no badge.
- Management rows render with live counts (`{d} doctors · {a} admins`) and the `Export · 30 days` audit summary.

---

## 4. Integration error queue

Purpose: let the admin see every synchronization event that did not process cleanly, understand why without unnecessary PHI exposure, and act — resolve an identity, inspect a payload, or retry. Governing principle (PRD §26, paraphrased by the screen subtitle): failed integrations must not silently lose clinical information.

### 4.1 Layout

**Header:**

- Back link: `‹ Organization` — 14 px, ink 60 % — returns to the dashboard.
- Title: `Legacy HIS — events` — Instrument Serif, 26 px. Format: `{integration name} — events`.
- Subtitle: `Failed events never silently drop clinical data.` — 13 px, ink 55 %. Fixed copy.

**Event cards** — one white card per problem event (14 px radius, `13px 15px` padding, 8 px gap). Each card: title row (event label 13.5 px semibold, right-aligned state badge — 10 px bold, pill), a reason line (12 px, ink 55 %), and, where the state affords one, a single action button.

The prototype's three demo events, verbatim:

| Event title | Badge | Reason line | Action |
|---|---|---|---|
| `Lab result LAB-102993` | `FAILED` (red pill) | `HOSP-928372 → ambiguous identity (2 candidate matches)` | `Resolve identity` — ink-filled pill button (12 px semibold, `7px 14px`, fully rounded) |
| `Discharge summary DS-5521` | `DEAD_LETTER` (neutral pill) | `Schema validation failed · missing encounter reference` | `View payload` — outlined pill button (white bg, `1px solid rgba(34,48,60,0.18)`) |
| `Imaging report IMG-8817` | `RETRYING` (amber pill) | `Endpoint timeout · attempt 3 of 5 · next retry 11:20` | none (automatic) |

Event title format: `{resource type} {source record id}` (e.g., `Lab result LAB-102993`). Reason lines are **PHI-minimized**: they name source identifiers, resource types, and technical causes — never patient names, demographics, or clinical values (PRD §26: understand the reason "without exposing sensitive information unnecessarily"; LLD: "reason (PHI-minimized)").

**Footer button** — full-width ink-filled button below the list: `Retry all recoverable` (14 px semibold, 12 px radius, `14px` vertical padding).

### 4.2 Queue states and per-state behavior

The full integration processing state machine is owned by [11-integrations.md](11-integrations.md) (PRD §26; architecture §11.2): `RECEIVED → VALIDATING → PROCESSED | REJECTED | RETRYING → FAILED → DEAD_LETTER`, with retries backed off exponentially and per-tenant dead-letter queues. The error queue surfaces the subset of states that need admin awareness or action:

| State | In queue? | Meaning | Badge style | Admin action | Resolution path |
|---|---|---|---|---|---|
| `RECEIVED` / `VALIDATING` | No | In-flight, healthy pipeline | — | — | Progresses automatically |
| `PROCESSED` | No | Persisted with provenance | — | — | Terminal (success) |
| `RETRYING` | Yes | Transient failure; automatic retry scheduled | Amber | None — informational. Card shows cause, `attempt {n} of {max}`, and `next retry {HH:MM}` | Succeeds → leaves queue; exhausts attempts → `FAILED` |
| `FAILED` | Yes | Needs a human decision (demo cause: ambiguous identity) | Red | `Resolve identity` when the cause is identity ambiguity (§4.3); counted by Retry-all (§4.5: re-evaluated on retry, re-fails while the blocker remains) | Resolved → reprocessed; unresolvable → admin rejection (Proposed, §4.6) |
| `REJECTED` | **Proposed:** yes, filtered/collapsed | Source's submission refused at authentication/authorization (e.g., unauthenticated source, disabled connector — a non-whitelisted resource type instead fails schema validation into DEAD_LETTER, [11-integrations.md](11-integrations.md) §8.2) | Red | Review reason | Terminal; source must resubmit |
| `DEAD_LETTER` | Yes | Cannot be processed as sent (demo cause: schema validation failure); parked in the per-tenant DLQ | Neutral gray | `View payload` (§4.4) | Corrected upstream and resubmitted, or replayed after a mapping/config fix (Proposed, §4.6) |

The retry/attempt limits and backoff schedule per integration come from that integration's error/retry configuration (PRD §21).

### 4.3 Resolve identity (FAILED · ambiguous identity)

The FAILED demo event carries an inbound record for hospital patient `HOSP-928372` that identity resolution matched to two candidate Atlas patients. Per PRD §22 and the LLD activity-flow constraint, this **must never auto-attach**: "Ambiguous matches should be routed for explicit resolution rather than automatically attaching clinical information to the wrong patient"; LLD: "Human resolution queue — «constraint» never auto-attach to wrong patient."

Behavior of the `Resolve identity` button:

- Routes the admin into the identity-resolution workflow for this event. The prototype confirms with the toast `Routed to identity resolution — never auto-attached` and does not depict the resolution screen itself.
- **Proposed — resolution screen:** shows the inbound record's PHI-minimized summary (resource type, source record ID, source hospital patient ID) beside the candidate Atlas patients with the matching attributes and confidence scores that produced the ambiguity (crosswalk → golden-record scoring, per the architecture pipeline). Admin actions: **Link** to one candidate, **Create new patient** (no candidate is correct), or **Reject event** (data is bad). Linking updates the identity crosswalk so future events for that hospital ID resolve automatically, then requeues the event for reprocessing.
- Every identity-linking operation is audited (PRD §22: "Every identity-linking operation should be auditable"), recording admin, tenant, event, chosen resolution, and timestamp.
- At no point between failure and explicit resolution is the clinical record attached to any patient or visible in any patient/doctor view.

### 4.4 View payload (DEAD_LETTER · secure viewer)

The DEAD_LETTER demo event is a discharge summary that failed schema validation (`missing encounter reference`). The `View payload` button opens the payload in a **secure viewer showing metadata only** — prototype toast: `Payload opened in secure viewer (metadata only)`.

- The viewer presents envelope and provenance metadata — the PRD §27 provenance set (source system, source organization, source record ID, source timestamp, record status) plus the declared resource type (the LLD's schema-validation envelope) and received-at time — and the structural validation errors (e.g., which required reference is missing). Provenance fields are shown verbatim as received (PRD §27).
- It does **not** render clinical content or patient demographics from the payload. This is the screen-level enforcement of PRD §26's minimal-exposure requirement.
- **Proposed:** raw-payload access (needed rarely, e.g., to file a vendor ticket) is a separate, explicitly elevated action behind a confirmation, and is written to the audit trail as a sensitive read.
- Payload viewing itself is an audited action (PRD §13 covers sensitive reads generally).

### 4.5 Retry all recoverable

The footer button requeues, in one tap, every event in the queue whose state permits automatic reprocessing. Prototype toast: `2 recoverable events queued for retry` — with the demo queue of three, the FAILED and RETRYING events are counted recoverable and the DEAD_LETTER event is not.

- **Recoverable** = `RETRYING` (retry now instead of waiting for the scheduled attempt) and `FAILED` events whose blocking condition can be re-evaluated. `DEAD_LETTER` events are excluded: a schema-invalid payload will fail identically until the payload or mapping changes, so they require §4.4/§4.6 handling. **Proposed:** a FAILED-ambiguous-identity event that is retried without resolution simply re-fails to the same state — retry never bypasses the no-auto-attach constraint; the definitive path for such events remains §4.3.
- The toast count reflects the actual number of events queued.
- Retries are safe to fire repeatedly: the pipeline's idempotency key `{tenant, source_system, source_record_id, version}` guarantees a duplicate that already processed is acknowledged and discarded, never double-persisted (architecture §11.2; LLD "Acknowledge & discard — source gets 200 — safe replay").
- The bulk retry is an audited admin action recording admin, tenant, integration, and affected event IDs. (**Proposed** detail; grounded in PRD §13.)

### 4.6 Edge cases

- **Proposed — empty queue:** when nothing is FAILED/RETRYING/DEAD_LETTER, show an empty state (`No failed events. Last sync {HH:MM}.`) and disable/hide `Retry all recoverable`. Reaching this state also returns the integration's dashboard card to healthy.
- **Proposed — event resolved elsewhere:** if an event leaves the queue (auto-retry succeeded, another admin resolved it) while the screen is open, the card is removed on next refresh; acting on a stale card returns a "no longer pending" notice rather than an error.
- **Proposed — retry exhaustion:** a RETRYING event that exhausts its configured attempts transitions to FAILED and its card gains the red badge; it is never dropped (the §4.1 subtitle is the invariant).
- **Proposed — permanently unprocessable:** closing out a DEAD_LETTER event (after upstream correction and resubmission, or with an explicit admin rejection recording a reason) is an audited terminal action; silent deletion of queue entries is not available to any role.

### Acceptance criteria

- The queue lists every event of the selected integration currently in RETRYING, FAILED, or DEAD_LETTER, each with resource-type + source-record-ID title, state badge, and a PHI-minimized reason line.
- No queue surface (list, reason lines, metadata viewer) renders patient names, demographics, or clinical values from an unprocessed payload.
- A FAILED ambiguous-identity event exposes `Resolve identity` and can only attach to a patient through explicit human resolution; there is no code path that auto-attaches an ambiguous match.
- Identity resolution outcomes update the crosswalk, requeue the event, and write an audit entry naming the admin and the chosen resolution.
- `View payload` on a DEAD_LETTER event shows envelope/provenance metadata and validation errors only.
- `Retry all recoverable` requeues exactly the RETRYING + eligible FAILED events, reports the count, and excludes DEAD_LETTER; replaying an already-processed event is idempotent (no duplicate records).
- No state transition ever discards an event without an audited terminal disposition.
- Queue state names and transitions match [11-integrations.md](11-integrations.md) exactly; this screen introduces no states of its own.

---

## 5. Data read and written

The admin app owns no clinical data. It reads and writes through `IAdminAPI`:

| Data | Access | Owner spec |
|---|---|---|
| Org profile (name, initials) | Read | [09-data-model.md](09-data-model.md) (tenant directory) |
| Staff counts / practitioner records | Read (write: §7.1 Proposed) | [09-data-model.md](09-data-model.md), [06-identity-auth.md](06-identity-auth.md) |
| Org patient count | Read | [09-data-model.md](09-data-model.md) |
| Integration configs and health | Read (write: §7.3 Proposed) | [11-integrations.md](11-integrations.md) |
| Integration events + processing states | Read; actions: resolve, retry, view metadata | [11-integrations.md](11-integrations.md) |
| Identity crosswalk entries | Write via explicit resolution only | [06-identity-auth.md](06-identity-auth.md), [09-data-model.md](09-data-model.md) |
| Audit log | Read/export only — never modify (tamper-resistant, PRD §13) | [09-data-model.md](09-data-model.md) |
| Access/publication policies | Read (write: §7.2 Proposed) | [07-consent-and-access-control.md](07-consent-and-access-control.md) |

Integration credentials and secrets are configured through the console but **never redisplayed** to any user after entry (PRD §21: "Integration credentials and secrets must never be exposed to ordinary users"; architecture: secrets live in the platform secrets manager).

---

## 6. Audit of admin actions

Every sensitive admin operation is itself audited (PRD §13). At minimum, the actions this spec defines that must produce audit entries:

```text
integration.event.retried          (single or bulk; event IDs enumerated)
integration.event.payload_viewed   (metadata viewer opened)
integration.event.rejected         (Proposed terminal disposition)
identity.link.resolved             (crosswalk updated by explicit resolution)
audit.exported                     (Proposed, §7.4; name per [07-consent-and-access-control.md](07-consent-and-access-control.md) §8.3)
staff.role.changed / staff.invited / staff.deactivated   (Proposed, §7.1)
integration.config.changed         (Proposed, §7.3)
break_glass.reviewed               (Proposed, §7.6; outcome + reviewer)
qr.static_issued / qr.static_revoked   (Proposed, §7.7)
```

Entries carry the PRD §13 metadata set: user, role, patient (where applicable), organization, timestamp, action, resource, access method, result, request/session information.

### Acceptance criteria

- Each admin action listed above yields exactly one audit entry with the PRD §13 metadata.
- Audit entries are append-only from the console's perspective; no admin UI affordance edits or deletes them.

---

## 7. Proposed: admin capabilities beyond the prototype

PRD §3.3 assigns the organization role more than the two prototype screens cover. Everything in this section is **Proposed** in its screen-level detail; the capability itself is PRD-mandated (citations inline). Suggested routes:

```text
/admin/staff                 Staff & roles
/admin/policies              Access policies
/admin/audit                 Audit log
/admin/integrations          Integration list + configuration
/admin/org                   Organization profile & configuration
/admin/compliance            Compliance review queue (break-glass sessions)
/admin/patients/static-qr    Static/printed QR issuance
```

### 7.1 Staff & roles (PRD §3.3 "Doctors and staff")

Target of the dashboard's `Staff & roles` row (`42 doctors · 8 admins`).

- List of practitioner and admin accounts in the tenant (from the tenant directory: practitioner records, org hierarchy). Columns: name, role (doctor / org-admin, extensible to platform-defined staff roles), specialty, facility, status (active / invited / deactivated).
- Actions: **Invite** (email invitation binding the account to the tenant IdP — architecture §16: admin "invites staff"), **Change role**, **Deactivate** (immediately revokes the account's access; sessions terminated per [06-identity-auth.md](06-identity-auth.md)).
- Doctor identity verification requirements (registration/licensing) belong to [06-identity-auth.md](06-identity-auth.md); this screen surfaces verification status, it does not bypass it.
- Deactivating a doctor never deletes records they authored; authorship and provenance are immutable (PRD §27).

### 7.2 Access policies (PRD §3.3 "Access policies"; dashboard row `Publication rules`)

- **Report publication rules** (PRD §24): per report type, one of the four PRD-defined modes — automatically published after finalization, published after verification, held for manual release, or restricted to healthcare professionals (the pipeline's "publication policy → hold: clinician-only visibility · manual release · restricted report types" branch in the LLD flow). UI: rule list per resource/report type with publish-mode selector.
- **Org-policy access** for records the organization itself created (architecture §7: grants can exist "by organization policy for records the org itself created"). Configuration of that policy's scope lives here; enforcement is the policy engine's ([07-consent-and-access-control.md](07-consent-and-access-control.md)).
- Policy changes are versioned and audited; they never widen access to records the org did not create.

### 7.3 Integration configuration (PRD §21; dashboard `Integrations` cards)

Per-integration configuration screen covering the PRD §21 list: organization identity, integration credentials (write-only), API endpoints, protocol (REST / FHIR / HL7 v2 / webhooks / secure file-based / scheduled sync — PRD §20), patient identifier mapping, supported clinical resources, report publication rules (linking to §7.2), notification preferences, synchronization mode, webhook configuration, integration status (enable/disable), and error/retry configuration (attempt limits, backoff, DLQ thresholds — the values §4.2 consumes).

- New or changed configurations are exercised against a **sandbox connector** before go-live (architecture §16: "test events flow through a sandbox connector before go-live").
- Full connector behavior, adapter contracts, and pipeline semantics: [11-integrations.md](11-integrations.md).

### 7.4 Audit log review & export (PRD §3.3 "Audit logs", §13; dashboard row `Export · 30 days`)

- Filterable, read-only view of the tenant's audit trail (filters: actor, patient, action, resource, access method, date range).
- **Export** produces a signed export of the selected window; the dashboard row fixes the default window at 30 days. Export is itself audited (§6).
- Tamper-resistance (append-only storage, integrity protection) is specified in [13-non-functional.md](13-non-functional.md).

### 7.5 Organization profile & configuration (PRD §3.3 "Organization profiles", "Organization-level configuration")

- Org display name, initials/branding shown in patient- and doctor-facing surfaces, facility list, org hierarchy (tenant directory).
- Tenant-level settings surfaced from the control plane where the org-admin may see them (tier, region/residency, quotas are visible but controlled by the platform, not editable by the tenant — architecture §16).
- Per-tenant feature flags are platform-managed; the console may display which features are enabled.

### 7.6 Compliance review queue (break-glass sessions; Architecture §16 flow D; [07-consent-and-access-control.md](07-consent-and-access-control.md) §6.4)

Every break-glass session is flagged for post-hoc review and must surface in an admin/compliance review queue (route `/admin/compliance`). [10-architecture.md](10-architecture.md) §14.4's acceptance criterion — "every break-glass session appears in a compliance review queue" — lands here.

- **Queue list**: one row per flagged session — doctor (name, specialty, facility), patient (Atlas ID; name per the reviewer's authority), reason code (`Unconscious patient` / `Critical care` / `Other` + free text), invocation timestamp, session duration, and a link to the session's full audit chain (every view and write under the emergency grant).
- **Review action**: the reviewer records an outcome — `justified` or `unjustified` — with a required note. An `unjustified` outcome routes to the misuse path of 07 §6.4: the org admin (or platform, for cross-tenant misuse) suspends the practitioner's Atlas access; the suspension is itself audited.
- Reviews have a defined turnaround (see [13-non-functional.md](13-non-functional.md) §9.2: compliance-notified events carry a review SLA); unreviewed sessions stay visibly pending in the queue.
- Every review action produces a §6 audit entry (`break_glass.reviewed`, Proposed) naming reviewer, session, outcome, and note. The queue is read-only over the underlying audit events — reviewing never modifies the session's audit trail.

### 7.7 Static / printed QR issuance (Architecture §8; [08-qr-subsystem.md](08-qr-subsystem.md) §8.3)

Printed/static fallback QRs exist for patients without phones; issuance, reissue, and revocation are an admin/enrollment concern delegated here by 08 §8.3 (route `/admin/patients/static-qr`).

- **Who may issue**: facility enrollment staff or org-admins (Proposed: a platform-defined `enrollment` staff role, manageable in §7.1). Issuance is tenant-scoped like every admin capability.
- **Identity verification at issuance**: the issuer verifies the patient's identity in person (government ID or ABHA verification, Proposed) and the verification method is recorded on the issuance. A static QR is never issued against an unverified identity claim.
- **Artifact**: the printed QR encodes a long-lived opaque pointer (not a 5-minute token — 08 §8.3); resolution always demands the additional identity check specified there.
- **Reissue / revocation on loss**: reporting a printed QR lost or stolen revokes the pointer immediately (subsequent resolves fail like any revoked token); reissue produces a new pointer. Old pointers are never reactivated.
- **Audit**: every issuance, reissue, and revocation writes an audit entry (`qr.static_issued` / `qr.static_revoked`, Proposed) with issuer, patient, verification method, and timestamp, visible in the patient's access log.
- API surface: admin-side issue/revoke routes in [12-api-and-interoperability.md](12-api-and-interoperability.md) §3.10; the doctor-side resolve route (`POST /v1/qr/static/resolve`) is in 12 §3.6.

### 7.8 Patient records created by the organization (PRD §3.3)

The PRD grants the org management of "patient records created by the organization." The prototype shows no admin surface for record content, and clinical record viewing is deliberately **not** an admin capability — admins operate on metadata, policy, and pipeline state, not on chart content. Managing org-created records means: publication policy (§7.2), integration reprocessing (§4), and provenance/lifecycle oversight. Any future admin-facing record operation must go through the policy engine and be audited like any other sensitive access.

### Acceptance criteria (for this section, once implemented)

- Each §7 capability is reachable from its dashboard row or the integrations section, scoped to the admin's tenant, and produces the §6 audit entries.
- Secrets entered in integration configuration are never returned by any read API or rendered in any UI after save.
- No admin screen renders clinical chart content; record oversight is metadata- and policy-level only.

---

## 8. Cross-references

| Topic | Spec |
|---|---|
| Integration pipeline, adapters, processing state machine, retry/backoff, DLQ | [11-integrations.md](11-integrations.md) |
| Identity crosswalk, golden record, admin/staff authentication | [06-identity-auth.md](06-identity-auth.md) |
| Policy engine, RBAC/ABAC, org-policy grants, publication enforcement | [07-consent-and-access-control.md](07-consent-and-access-control.md) |
| Entities: tenant, practitioner, integration event, audit entry | [09-data-model.md](09-data-model.md) |
| `IAdminAPI` surface | [12-api-and-interoperability.md](12-api-and-interoperability.md) |
| DLQ depth alarms, integration lag SLOs, audit tamper-resistance | [13-non-functional.md](13-non-functional.md) |
| Meridian tokens, card/badge/toast styles | [01-design-system.md](01-design-system.md) |
| Tenancy model and platform control plane | [10-architecture.md](10-architecture.md) |
