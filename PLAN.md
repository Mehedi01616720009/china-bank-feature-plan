# PLAN.md — CAI Coach (Module B) + CCC Voice AI (Module C)

## 0. Scope Recap
Two connected deliverables for Chinabank (MyCBC):
- **Module B — CAI Coach**: authenticated, in-app AI orchestration layer (chat + guided UI + transactions) inside the Backbase-based MyCBC app.
- **Module C — CCC Voice AI**: voice bot / agent-assist for the contact center (Genesys), sharing identity and possibly the NLU/orchestration core with Coach.

Both modules sit on top of the same backend orchestration service; Voice is a second channel into it, not a separate product.

---

## 1. Architecture Decision — Orchestration Pattern

**Chosen pattern: Hybrid, API-driven-first.**

- **Primary path — API-driven UI Intent Schema**: Coach backend returns a versioned JSON schema (cards, forms, actions, confirmations). MyCBC's Backbase renderer maps schema → native Backbase components. This avoids vendor lock-in into a proprietary widget and keeps design-system ownership with CBC (addresses PROBLEMS #1, #3, #4).
- **Secondary path — thin embeddable conversation shell**: a small SDK/widget only for the conversation panel itself (streaming chat thread, input box, launcher entry point) — the one piece that benefits from a pre-built, tested component instead of reinvention. This shell renders content *using* the UI intent schema, not its own bespoke cards.
- **Rationale for hybrid over pure-SDK**: pure SDK maximizes vendor lock-in and duplicates Backbase's design system; pure schema-only would force CBC to hand-build the conversation UI too, which is unnecessary rework. Hybrid isolates the lock-in surface to one low-risk component.

**UI Intent Schema (PROBLEMS #3)**: versioned (`schema_version` field, additive-only changes within a major version), covers: text blocks, intent-confirmation card, action card, form spec, review/confirmation screen, step-up prompt, receipt card, "why" module, feedback control — one schema entry per component in the B.7 inventory.

---

## 2. High-Level System Architecture

```
MyCBC App (iOS/Android/Web, Backbase)
   │  renders UI-intent-schema JSON via Backbase components
   │  conversation shell (thin SDK) for chat thread only
   ▼
API Gateway (CBC-owned) ── OAuth2/OIDC/JWT, mTLS
   ▼
Coach Orchestration Service (our build)
   ├─ Session/Journey State Manager   (Redis — cross-task/tab continuity)
   ├─ NLU/Intent Layer                (entity extraction, intent, language ID)
   ├─ Dialogue/Orchestrator Core      (state machine per journey, shared with Voice)
   ├─ RAG/Insights Engine             (retrieval + grounding + citation)
   ├─ Explainability Logger           (basis object → audit store)
   └─ Action Executor                 (idempotent calls, only via EIP gateway)
   ▼
EIP Gateway (CBC-owned broker) ── least-privilege scopes per call
   ├─ Core Banking / Cards / Payments
   ├─ CRM (Dataverse/D365 via Entra ID/MSAL)
   ├─ Identity Provider (OIDC, step-up OTP/biometric)
   └─ ECMS / Knowledge Repositories (RAG source content)

Event Bus / Webhooks (async: app status, transfer completion, NBO triggers)
   → common event envelope → Coach Orchestration Service + downstream subscribers

Voice Channel (Module C)
   Genesys ──(telephony events)── Voice Gateway ── STT/TTS ── same
   Dialogue/Orchestrator Core + Session Manager as above, voice-adapted
   (no visual step-up → OTP-by-voice / KBA / telephony-linked auth)
   → escalation: transcript + context handed to live agent, no repetition
```

Key boundary rule (PROBLEMS #5, #21): **Coach never calls Core Banking/CRM/Cards directly.** Everything routes through the EIP gateway, which is the only place scopes are enforced and the only path money moves through.

---

## 3. Core Sequence Flows

### Flow 1 — Account Overview → Transfer
1. User opens Coach (already authenticated in MyCBC) → session context loaded (profile, holdings, recent activity — least-privilege scoped).
2. `GET /accounts` via EIP → render account overview + insights/action cards (UI intent schema).
3. User types "Transfer ₱5,000 to Juan" → NLU extracts intent + entities → intent-confirmation card rendered (editable entities).
4. User confirms/edits → pre-filled transfer form (review + confirmation screen: fees/limits/disclosures injected).
5. User confirms → step-up auth challenge (OTP/biometric) triggered inline, resumable on success/failure/lockout.
6. On success → `POST /transfers` with `Idempotency-Key` → executed via Core Banking/Payments through EIP.
7. Confirmation + reference shown → receipt/audit-reference card rendered → full event logged to audit trail.

### Flow 2 — Unauthenticated Assist → Coach Continuity
1. User asks unauthenticated CAI Assist a question → general guidance given.
2. Assist offers "Continue in MyCBC" → issues a **resume token**: signed, short-TTL, contains a journey reference id only (no PII, no balances, no entities) — not replayable, not forgeable.
3. User taps continue → OIDC login.
4. Post-login, authenticated Coach session exchanges resume token → looks up journey state server-side (state itself never left the backend) → reconstructs the guided flow (e.g., transfer with secure pre-fill) exactly where the user left off.

### Flow 3 — Insights/NBO → Apply
1. `GET /insights` → eligible signals fetched from CRM/Core → insight card with recommended product.
2. "Why this offer?" → rationale + "basis" object + disclosures + opt-out rendered (explainability module).
3. "Apply" → OAuth-scoped application draft created in D365/Dataverse → draft id returned.
4. Pre-filled application form → user reviews/edits/submits (+ attachments) → confirmation, reference, tracking info shown; draft is resumable if abandoned.

### Flow C — Voice (Module C, mirrors Flow 1 without visual UI)
1. Call routed via Genesys → bot answers → identity verification (voice OTP / KBA / telephony-linked auth) before any protected disclosure or servicing action.
2. Multi-turn dialogue manages the same intents as Coach (shared NLU core where possible) — confirmation is spoken back explicitly before any action ("You want to transfer ₱5,000 to Juan — confirm?").
3. On low-confidence STT/misrecognition → clarification loop, not silent guess.
4. On escalation → transcript + structured context (intent, entities already confirmed, journey stage) handed to agent desktop — agent doesn't re-ask what's already known; only data-minimized transcript is persisted per privacy rules.

---

## 4. Workstreams (mapped from PROBLEMS.md)

| # | Workstream | Key problems covered |
|---|---|---|
| 1 | Contract-first API layer (OpenAPI, webhooks section) | 2, 6, 79 |
| 2 | UI Intent Schema design + versioning | 3, 10, 24 |
| 3 | Session/Identity/Security (OIDC, step-up, resume token, idempotency) | 15–19, 68, 70 |
| 4 | NLU/Orchestration core (intent, entities, code-switching, state mgmt) | 8, 9, 11 |
| 5 | RAG + Explainability (grounding, citations, basis logging) | 12–14, 34 |
| 6 | Frontend/Mobile (conversation shell, form renderer, cards, a11y) | 22–29 |
| 7 | Data/Latency/Reliability (caching, provenance, degradation) | 30–33 |
| 8 | Governance/Audit/Compliance (consent, NIST/ISO mapping, DPA/IRR) | 34–39 |
| 9 | Localization & Accessibility (en/fil/Taglish, WCAG 2.2 AA) | 40–42 |
| 10 | Testing & Acceptance (B-AT-01..05, persona coverage) | 43–45 |
| 11 | Voice AI (Genesys, STT/TTS, voice identity, handoff) | 46–51 |
| 12 | Delivery/Program mgmt (WBS, sprints, telemetry, BAU) | 52–55 |

---

## 5. Phased Delivery Plan (2-week sprints)

### Phase 1 — Foundation (Sprints 1–3)
- UX discovery per persona (6 personas from B.3), clickable prototypes.
- Finalize orchestration pattern (this doc, Section 1) + EIP gateway boundary + scope model.
- OpenAPI contract skeleton for all mandatory journeys (accounts, transfers, insights, apply, cards) including webhooks section.
- UI Intent Schema v1 draft + Figma-to-schema mapping with Backbase design tokens.
- Security model: OAuth2/OIDC/JWT flow design, step-up auth flow design, resume-token spec.
- **Exit criteria**: CBC sign-off on journeys, orchestration pattern, UI approach, and security model.

### Phase 2 — MVP (Sprints 4–11)
- Coach shell (conversation panel) + launcher entry point.
- Account overview journey (live, SLA-tested).
- Insights v1 (RAG pipeline, basic explainability, no unsafe-claims guardrail).
- Transfer + Bill Pay journeys, full step-up auth, idempotent execution, receipts.
- Audit trail v1 (consent capture, confirmations, basis objects).
- Telemetry baseline stood up (must exist before Expansion, not bolted on later — problem #54).
- **Exit criteria**: mandatory journeys pass B-AT-01, B-AT-02, B-AT-03.

### Phase 3 — Expansion (Sprints 12–21)
- Apply flow (NBO → CRM draft → submission), Cards services (lock/unlock, status).
- Disputes/service request journeys.
- Assist → Coach continuity (Flow 2) fully implemented + regression-tested (cross-system, high silent-breakage risk — problem #45).
- RAG provenance UI ("why" module with citations) hardened.
- Observability hardened (structured logs, tracing across EIP hops).
- Voice AI (Module C) MVP: Genesys integration, STT/TTS tuned for PH accents/code-switching, voice identity verification, transcript handoff.
- **Exit criteria**: mandatory + optional requirements pass; B-AT-04, B-AT-05 pass.

### Phase 4 — Optimization (Ongoing)
- NBO funnel experimentation, A/B on entry-point placement.
- Model risk monitoring: drift dashboards, ongoing AI RMF govern/measure/manage cadence.
- BAU handover: owning team + runbook for ongoing monitoring (not a one-time delivery).
- **Exit criteria**: SLA/KPI stabilization, signed-off BAU handover.

---

## 6. Open Decisions to Confirm with CBC
1. Web support for Coach — optional per RFP; confirm if in scope.
2. Whether Voice AI and Coach share one NLU/orchestration core (recommended) or run separate stacks — affects Phase 3 sequencing.
3. Non-blocking vs. blocking confirmation decision table ("what counts as high-risk") — needs joint product/engineering sign-off before Form Renderer + Review Screen work starts in MVP.
4. Final identity-assurance method for voice channel (voice OTP vs. KBA vs. telephony-linked) — blocks Voice MVP identity work.
5. Genesys integration depth (routing + handoff only vs. full agent-assist) — affects Module C WBS sizing.
