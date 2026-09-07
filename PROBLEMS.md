# PROBLEMS — Engineering Tasks & Challenges to Solve
Derived from RFP Module B (CAI Coach) and Module C (CCC Voice AI). Organized by
engineering domain — this is the "so what does my team actually have to build/solve"
view of FEATURES.md.

---

## 1. Architecture & Integration Pattern Decisions
1. **Choose the orchestration pattern**: front-end SDK/embeddable widget vs. API-driven
   "UI intent schema" composition vs. a hybrid — and justify the trade-offs (vendor
   lock-in vs. schema-governance overhead).
2. Design a **contract-first API layer** using OpenAPI as the source of truth (specs
   must exist *before* implementation, including a webhooks section).
3. Design the **UI intent schema** (if API-driven pattern chosen): a versioned,
   forward-compatible JSON schema for cards/forms/actions that MyCBC's Backbase
   renderer can consume without vendor-specific code.
4. Define how Coach **plugs into the existing Backbase design system** — token
   mapping, component library reuse vs. custom components, Figma-to-code alignment.
5. Design the **Enterprise Integration Platform (EIP) gateway boundary**: what Coach
   is allowed to call directly vs. what must be brokered, and how least-privilege
   scopes are enforced per call.
6. Define the **event bus / webhook architecture** for async flows (app status,
   transfer completion, NBO triggers), including delivery guarantees, retries, and
   a common event envelope format for interoperability.
7. Decide **build vs. reuse** for each of the 10 UI components in the inventory
   table, per platform (iOS/Android/web).

## 2. Conversational AI / NLU / Orchestration Engine
8. Build an **intent recognition + entity extraction** pipeline that reliably maps
   free text ("Transfer ₱5,000 to Juan") into structured transfer intents.
9. Solve **multi-turn state management**: Coach must preserve journey state across
   tasks/tabs/app backgrounding without re-prompting the user (B-AT-03).
10. Build the **"intent confirmation" correction loop** — user must be able to edit
    extracted entities and have the system re-render updated cards, not restart
    the flow.
11. Solve **language identification and code-switching** ("Taglish") in real time,
    tagging content per BCP 47 / RFC 5646 and routing to correct locale content.
12. Design the **RAG pipeline** for Insights, "Why" explanations, and Financial
    Coaching: content ingestion from ECMS/knowledge repositories, versioning,
    retrieval, and citation of the "data basis" for every claim.
13. Ensure **no unsafe/unsupported claims** are surfaced from insights — needs a
    grounding/verification layer between the model and the UI, not just a
    disclaimer.
14. Build the **explainability logging pipeline**: every recommendation must log a
    machine-readable "basis" object, retrievable for audit (B-06, B-AT-04 — 20
    sample recommendations must all have rationale + logged basis).

## 3. Session, Identity & Security
15. Implement **authenticated session context propagation** (profile, holdings,
    recent activity) into Coach with least-privilege scoping per B-02 — avoid
    over-fetching or over-exposing customer data to the AI layer.
16. Implement **OAuth 2.0 / OIDC / JWT** token issuance, refresh, and validation
    across MyCBC App ↔ Coach ↔ API Gateway ↔ downstream systems.
17. Implement **step-up authentication** (OTP/biometric) as a reusable, interruptible
    challenge/response flow that Coach can trigger mid-conversation and resume from
    cleanly on success or failure/lockout.
18. Design the **unauthenticated Assist → authenticated Coach handoff**: create a
    "resume token" that carries *zero* sensitive data, survives login, and lets
    Coach reconstruct journey state post-auth (Flow 2) — this is a nontrivial
    state-serialization + security problem (token must not be replayable/forgeable
    and must expire).
19. Implement **idempotent transaction execution** for transfers (Idempotency-Key
    header) to prevent double-submission from retries or flaky mobile networks.
20. Integrate with **Microsoft Entra ID / MSAL** for Dataverse/D365 CRM access —
    handle app registration, token acquisition, and scoped Dataverse calls.
21. Enforce that **all money-moving / sensitive actions route only through
    CBC-approved APIs** — prevent any "shortcut" path from the LLM/orchestrator
    directly to core systems.

## 4. Frontend / Mobile Engineering
22. Build a **mobile-first, tablet-responsive** Coach UI (iOS/Android native or
    cross-platform) meeting WCAG 2.2 AA, including screen-reader, dynamic-type,
    and reduced-motion support.
23. Implement **streaming chat responses** with editable user messages, retry, and
    controlled copy/share (respecting data-sensitivity rules on what can be shared).
24. Implement the **progressive disclosure pattern** UI-wide (collapsed summary →
    expandable "Show fees" / "Why recommended" / "See breakdown") consistently
    across all cards.
25. Implement **non-blocking (snackbar/toast) vs. blocking (modal) confirmation**
    rules — needs a shared design-system decision table for "what counts as
    high-risk" that engineering and product must jointly own.
26. Build a **generic, config-driven form renderer** (validation, masks, helper
    text, error states) that can render bank-grade forms (transfer, bill pay,
    applications) from schema rather than being hand-coded per journey.
27. Build the **review + confirmation screen** pattern with fee/limit/disclosure
    injection points that compliance content can update without a code release.
28. Build the **receipts/audit reference** screen with transaction ID, status, and
    controlled share/download rules (must not leak sensitive data via share sheet).
29. Solve **entry-point discoverability** (FAB/tab/header icon) with A/B-testable
    placement and analytics instrumentation, permission-aware (don't show Coach
    features the user isn't entitled to).

## 5. Data, Latency & Reliability
30. Meet **p95 latency SLAs** for account overview while aggregating multiple
    backend systems (core banking, cards, data services) — needs caching /
    parallel-fetch / graceful-degradation strategy.
31. Guarantee **data correctness vs. system of record** — no stale or
    eventually-consistent balance being shown as authoritative without
    provenance labeling.
32. Design **provenance metadata** attached to every computed insight/number shown
    (source system, as-of timestamp) for both UX display and audit logging.
33. Design **error handling that's "safe and transparent"** — every orchestration
    step needs a defined failure UX (timeout, downstream 5xx, partial data) rather
    than silent failure or raw error leakage.

## 6. Governance, Audit & Compliance
34. Build a **full audit trail** covering: consent capture (onboarding), every
    sensitive-action confirmation + step-up event, every recommendation's
    rationale/basis, and dispute-relevant interaction logs.
35. Implement **consent and opt-out management** (Coach onboarding consent, NBO
    opt-out) as a persisted, queryable customer state — not just a UI toggle.
36. Map controls to **NIST/ISO (or equivalent) security & privacy frameworks** and
    produce evidence for the B-AT-05 security review.
37. Align the model-risk program to **AI RMF govern/measure/manage** practices —
    requires ongoing monitoring, not just launch-time review (ties into
    "Optimization" phase: drift dashboards, model risk monitoring).
38. Implement **Data Privacy Act / IRR compliance**: minimal data collection by
    design, privacy notices at point of collection, and a **breach-notification
    timeline workflow** wired into incident response.
39. Ensure **BSP consumer-protection-aligned disclosures** (fees, terms, key risks)
    are enforced structurally in Apply/NBO flows — not optional copy that can be
    skipped.

## 7. Localization & Accessibility
40. Build a **configurable copy/content system** supporting English, Filipino, and
    Taglish without hardcoding strings, including locale-aware number/date/currency
    formatting.
41. Validate **WCAG 2.2 AA compliance** end-to-end (not just component-level) —
    contrast, focus order, touch-target size, screen-reader labeling for dynamic
    AI-generated content specifically (a known hard problem: labeling
    streaming/partial chat content accessibly).
42. Design UX specifically for the **senior/accessibility persona** (large text,
    reduced cognitive load, forgiving error recovery) as a first-class tested
    persona, not an afterthought.

## 8. Testing, QA & Acceptance
43. Build automated test coverage mapped to every acceptance test: B-AT-01
    (orchestration + SLA), B-AT-02 (step-up + audit record), B-AT-03 (session
    continuity across task switches), B-AT-04 (explainability across 20 sample
    recommendations), B-AT-05 (security control evidence), A-AT-08-equivalent for
    card lock/unlock identity verification.
44. Build **persona-based UX test coverage** (per 4.2.2) — test plans/rationale
    per persona group, not a single generic test pass.
45. Build regression tests for the **cross-module continuity flow** (Assist →
    Coach) since it spans two systems, an identity provider, and token exchange —
    high risk of silent breakage.

## 9. Voice AI (Module C) Specific Problems
46. Integrate with the **contact center platform (Genesys)** for call routing,
    escalation, and warm agent handoff — including telephony-side event handling.
47. Build/tune **STT/TTS models for Philippine-accented English and Filipino**,
    including **code-switching mid-utterance**, which most off-the-shelf STT
    struggles with.
48. Build **multi-turn dialogue management** for voice (no visual UI to fall back
    on) — needs robust confirmation-by-voice patterns and graceful
    misrecognition recovery.
49. Implement **voice-channel identity verification** before disclosing protected
    info or allowing servicing actions — solve this without visual step-up UI
    (e.g., voice OTP, knowledge-based auth, or telephony-linked auth).
50. Implement **transcript + context handoff to live agents**, ensuring the agent
    sees exactly what the bot discussed (no repetition) while respecting
    data-minimization/privacy rules on what's persisted in the transcript.
51. Decide whether Voice AI and Coach **share the same orchestration/NLU core** or
    are separate stacks — reuse vs. duplication trade-off, and how identity/session
    state is shared between voice and app channels if a customer switches channels
    mid-journey (an extension of the Flow 2 problem to voice).

## 10. Delivery / Program-Level Problems
52. Produce a **detailed WBS and 2-week sprint plan** across Foundation → MVP →
    Expansion → Optimization phases with explicit exit criteria per phase.
53. Sequence work so the **MVP truly ships a working, mandatory-journey-complete
    product** (account overview, insights v1, transfer + bill pay with step-up,
    telemetry) before expansion features — avoid scope creep into Foundation.
54. Stand up a **telemetry baseline** early (MVP phase) so later optimization
    (drift dashboards, KPI stabilization) has data to work from — this is a
    dependency that must be built, not bolted on later.
55. Establish a **BAU (business-as-usual) handover process** for the Optimization
    phase — ongoing model monitoring/experimentation needs an owning team and
    runbook, not just a one-time delivery.
