# FEATURES — Modular AI Platform RFP (Chinabank)
Source: Module B (4.2 CAI Coach) and Module C (4.3 CCC Voice AI)

---

## MODULE B: CAI COACH

### B.1 Core Concept
1. Coach is an **authenticated capability embedded in MyCBC** — not a public/anonymous feature.
2. Goes "beyond chat": functions as an **AI-powered orchestration layer**, not a standalone chatbot.
3. Delivers banking journeys through a **hybrid experience**: conversation + guided UI + transactional execution.
4. Must let customers **quickly understand their financial position**.
5. Must let customers **complete high-frequency tasks with fewer steps**.
6. Must deliver **compliant, explainable recommendations / next-best actions**.
7. Must preserve **customer control** and **secure confirmations** for sensitive operations.
8. Must act as a **journey orchestrator** that preserves **state across a multi-step flow**.
9. Must **generate structured UI responses** (not just free text).
10. Must **execute actions through well-defined API contracts**.
11. Must render **consistent UI** and **handle errors safely and transparently**.
12. **OpenAPI** is the mandated standard for contract-first API definition and discoverability.

### B.2 Assumptions & Design Constraints (4.2.1)
13. **Mobile-first UI** (iOS/Android) with **responsive tablet** behavior.
14. **Web support is optional** unless CBC explicitly requires web-embedded Coach.
15. **Accessibility**: conform to **WCAG 2.2 AA** for all user-facing Coach UI, across device types (including mobile).
16. **Localization**: support **English + Filipino + mixed-language ("Taglish")** content.
17. Localization delivered via **configurable copy** and **locale-aware formatting**.
18. **Language identification** must follow **IETF language tagging (BCP 47 / RFC 5646)**.

### B.3 UX Personas Supported (4.2.2) — minimum required
19. Retail — **Digital natives / frequent transactors**: speed, shortcuts, low friction.
20. Retail — **Mass retail / occasional users**: clarity, guided steps, confidence.
21. Retail — **Affluent / relationship-oriented**: high trust, explainability, concierge-like flows.
22. Retail — **Seniors / accessibility needs**: large text, reduced cognitive load, error recovery.
23. **Power users**: manage multiple products and frequent payments.
24. **Operations, risk & compliance users**: need auditability, content governance, consumer-protection checks, dispute-handling pathways.
25. Vendor must provide **persona-based UX rationale and test coverage** for each group.
26. Designs must **set expectations, support efficient interaction, and allow recovery when the system is wrong** — across all personas.

### B.4 Prioritized Journeys — all Mandatory (4.2.3)
27. **Onboarding to Coach** — discover Coach, understand capabilities/limits, capture consent & preferences. Touchpoints: Identity provider, MyCBC shell. Acceptance: disclosure + consent captured; capabilities/limits displayed; user can opt out; full audit log.
28. **Authentication** — log in to access account via Authentication layer. Acceptance: login using registered data.
29. **Account overview** — view consolidated balances/recent activity. Touchpoints: Enterprise Integration Platform, Core banking/cards, data services. Acceptance: data correct vs. system of record; p95 latency per SLA; provenance shown for computed insights.
30. **Insights and alerts** — understand spending/fees/anomalies with explainable insights. Touchpoints: Analytics layer, knowledge base. Acceptance: explanations provided; dismiss/feedback supported; no unsafe claims; logs include "basis" metadata.
31. **Transfer funds** — pre-fill + execute transfer securely. Touchpoints: Enterprise Integration Platform, Payments/core, identity provider. Acceptance: review + confirmation + step-up auth for sensitive actions; idempotent execution; receipt shown.
32. **Bill pay / start-continue product application** — pre-filled application data. Touchpoints: Enterprise Integration Platform, CRM (D365), underwriting/workflow. Acceptance: pre-fill from profile; explicit user review; submission acknowledgment; save/resume supported.
33. **Cards Services** — card lock/unlock, application status inquiry. Touchpoints: Enterprise Integration Platform, Core Cards. Acceptance: confirmation + new status provided.
34. **Financial coaching** — set goals, follow guided coaching plan. Touchpoints: Analytics models, content KB. Acceptance: goals persisted; nudges explainable; user can correct assumptions/preferences.
35. **Next Best Offer (NBO) flow** — receive recommendation, understand rationale, start application. Touchpoints: CRM (D365), product catalog. Acceptance: rationale + disclosures shown; opt-out available; conversion tracked end-to-end.
36. Journeys must satisfy **consumer protection / financial consumer protection law** requirements: disclosure/transparency, privacy, complaint handling — specifically in onboarding disclosures, dispute journeys, and product/pricing presentation (Apply, NBO).

### B.5 Required Interaction Models (4.2.4)
37. **Conversational + structured UI hybrid**: natural-language input; response combines (a) short text and (b) actionable UI components (cards, forms, review screens).
38. Must communicate **what the system can do** and enable **correction and recovery** (human-AI interaction guidance).
39. **Progressive disclosure**: summarize first, reveal details on demand ("See breakdown", "Why recommended", "Show fees").
40. Progressive disclosure must **reduce interruption/cognitive load** and avoid unnecessary modal interruptions.
41. **Micro-interactions for feedback**: non-blocking confirmations (snackbar/toast) for low-risk actions.
42. **Modal confirmations** reserved for high-risk / critical actions only.
43. **Form-centric orchestration**: bank-grade validation, formatting, and review steps for transfers, bill pay, applications — including confirmation screens and error recovery.

### B.6 UI Components & Design System (4.2.5)
44. Coach UI must **use CBC's existing Backbase-based design system**, OR
45. Provide a **compatible design-system strategy** integrating with Backbase design tokens/component libraries across web/iOS/Android, with **documentation and Figma alignment**.
46. All user-facing Coach UI must meet **WCAG 2.2 AA**, addressing **mobile accessibility** per W3C guidance.

### B.7 UI Component Inventory (4.2.6) — vendor must state build-vs-reuse, platforms, customization, accessibility
47. **Coach launcher entry point** — FAB/button/tab/header icon; permission-aware, A/B configurable placement, analytics hooks.
48. **Conversation panel** — chat thread with timestamps; streaming responses, editable user messages, retry, copy/share where allowed.
49. **Intent confirmation card** — "You want to transfer…" pattern; editable entities, confidence indicator, "change" controls.
50. **Action cards** — cards with CTA(s); support multiple actions, disable when policy blocks, show prerequisites.
51. **Form renderer** — inline form/bottom sheet/modal; field validation, input masks, helper text, error states.
52. **Review + confirmation screen** — full-screen review with "Confirm" CTA; fees/limits display, disclosures, added friction for risky actions.
53. **Step-up authentication prompt** — modal/bottom sheet; supports OTP/biometric hooks, clear fallback and lockout messaging.
54. **Receipts and audit references** — receipt page/card; transaction ID, timestamp, status, share/download rules.
55. **"Why" / explainability module** — expandable section; rationale text + data basis + source citations for KB content.
56. **Feedback and correction controls** — thumbs up/down, "Report issue"; captures user correction, routes to analytics and KB ops.

### B.8 Consumer Protection & Privacy UX (4.2.7)
57. Clear **disclosures/transparency** for product features, terms, fees, key risks in "Apply" and NBO journeys — aligned to financial consumer protection law and BSP consumer protection framework.
58. **Data privacy notices**, **consent capture** where required, and **minimal data collection** — aligned to the **Data Privacy Act and its IRR**.
59. Compliance with **breach notification timelines** in the IRR for relevant incidents.

### B.9 Functional Requirements (4.2.8) — all Mandatory
60. **B-01**: Integrate with Backbase MyCBC front-end (vendor proposes: SDK/widget, embedded web component, or API-driven UI composition).
61. **B-02**: Support authenticated session context (customer profile, product holdings, recent activity), subject to **least-privilege access controls**.
62. **B-03**: Support journey orchestration — guided navigation, pre-filled forms, step-by-step task assistance, execution of approved actions via APIs.
63. **B-04**: Support explicit confirmations and **step-up authentication** (OTP/biometrics where applicable) before sensitive actions.
64. **B-05**: (duplicate of orchestration requirement) guided navigation, pre-filled forms, step-by-step assistance, approved-action execution via APIs.
65. **B-06**: Provide **explainable outputs** for recommendations/insights ("why this suggestion"), log the basis for audit — aligned to **AI RMF** trustworthy-AI governance.
66. **B-07**: Provide personalized financial health insights (spending analysis, savings goals, reminders) using approved analytics models and transparent assumptions.
67. **B-08**: May provide **next-best-offer nudges**, subject to CBC approvals, disclosures, and opt-outs.

### B.10 Constraints & Controls
68. Transactions must be executed **only** through CBC-approved systems/APIs — Coach must not "shortcut" execution via non-approved channels.
69. All model-driven guidance must be **governed, measurable, and monitored** for risk/performance (AI RMF govern/measure/manage).
70. Authentication/federation/token handling must use **OAuth 2.0, OpenID Connect, JWT**, over **TLS**.

### B.11 Module Acceptance Tests (4.2.9)
71. **B-AT-01** In-app orchestration ("show my account overview") — Coach fetches/displays via approved APIs; latency/correctness within agreed SLA.
72. **B-AT-02** Sensitive action (transfer/pay bill) — explicit confirmation + step-up auth required; produces audit record with transaction reference.
73. **B-AT-03** Session continuity — user switches between tasks; Coach retains state, avoids repeated prompts.
74. **B-AT-04** Explainability — for 20 recommendations, Coach provides rationale + logged basis.
75. **B-AT-05** Security review — vendor evidence of alignment to security/privacy control cataloging and governance (e.g., NIST/ISO control mapping).

### B.12 Orchestration & Integration Patterns (4.2.10.1)
76. Vendor must propose **one pattern (or hybrid)**, with trade-offs:
    - **Front-end SDK / embeddable component pattern**: vendor SDK/widget in MyCBC renders UI, manages local state, calls CBC APIs via gateway.
    - **API-driven UI composition pattern**: Coach returns a structured **"UI intent schema"** (cards/forms/actions); MyCBC renders via CBC-owned components (reduces vendor lock-in, needs stronger schema governance).

### B.13 Eventing, Webhooks & Data Flow (4.2.10.2)
77. Support an **event model + delivery mechanism** (event bus and/or webhooks) for async orchestration (application status updates, transfer completion events, NBO campaign triggers).
78. Recommended alignment to a **common event format** for interoperability (e.g., CloudEvents-style).
79. **Webhook support documented via OpenAPI** (webhooks field) or equivalent.

### B.14 Required Integration Touchpoints (4.2.10.3)
80. **Backbase/MyCBC shell** and design-system foundations (design tokens, component libraries).
81. **CRM**: Microsoft Dataverse / Dynamics 365 via OAuth-based access (Microsoft Entra ID where applicable) — app registration + MSAL usage patterns.
82. **Core banking and payments** (CBC systems of record) via CBC-approved APIs.
83. **ECMS / knowledge repositories** for RAG content ingestion and versioning.

### B.15 Required Sequence Flows (4.2.10.4–4.2.10.6)
84. **Flow 1 — Account overview → Transfer**: Open Coach (authenticated) → start session/context → GET /accounts → render insights/actions → NL transfer request → intent+entities extraction → pre-fill transfer form → user confirms → step-up auth challenge (OTP/biometric) → step-up success token → POST /transfers with **Idempotency-Key** → execute via Core Banking/Payments → confirmation + reference → show receipt + audit reference. Must support OAuth/OIDC session handling end-to-end.
85. **Flow 2 — Unauthenticated Assist → Coach continuity**: user asks unauthenticated CAI Assist a question → Assist gives general guidance + "Continue in MyCBC" → creates a **resume token containing no sensitive data** → user taps continue → OIDC login → authenticated session → resume with token + authenticated context → Coach restores journey state and shows the guided flow (e.g., continue transfer with secure pre-fill). No sensitive data exposed pre-authentication.
86. **Flow 3 — NBO / Insights → Apply**: View insights → request personalized insights → GET /insights → fetch eligible signals from CRM/Core → insight card with recommended product + "Why?" affordance → tap "Why this offer?" → rationale + disclosures + opt-out shown → tap "Apply" → start application (OAuth) → create application draft in D365/Dataverse → draft ID → pre-filled application form → user reviews/submits → submit application + attachments → confirmation + reference + tracking info shown.

### B.16 Delivery Plan / Phases (4.2.11) — 2-week sprints assumed
87. **Foundation** (2–3 sprints): UX discovery, IA, clickable prototypes, architecture + integration plan, design-system alignment. Exit: CBC sign-off on journeys, UI approach, security model.
88. **MVP build** (6–8 sprints): Coach shell + conversation UI, account overview, insights v1, transfer + bill pay with step-up auth, telemetry baseline. Exit: mandatory journeys pass.
89. **Expansion** (6–10 sprints): Apply flows, disputes/service requests, Assist→Coach continuity, RAG provenance UI, hardened observability. Exit: mandatory + optional requirements pass.
90. **Optimization** (ongoing): NBO funnels, experimentation, model risk monitoring, drift dashboards, continuous improvement. Exit: SLA/KPI stabilization, BAU handover.
91. Vendor must provide a **detailed Work Breakdown Structure (WBS)** and recommend phase/sprint activity/timeline.

---

## MODULE C: CCC VOICE AI

### C.1 Core Concept
92. Provides **voice-based customer support automation** (voice bot) and/or **agent-assist**.
93. Must support **secure escalation to live agents with conversation context** preserved.

### C.2 Functional Requirements (4.3.1) — all Mandatory
94. **C-01**: Integrate with the Bank's **contact center environment** (e.g., **Genesys**) for call routing, escalation, and agent handoff.
95. **C-02**: Support **real-time STT/TTS**, intent recognition, and **multi-turn dialogue management**, suitable for **Philippine accents and code-switching**.
96. **C-03**: Support **secure identity verification** prior to disclosing protected information or enabling servicing actions; identity assurance per well-defined authentication/federation requirements.
97. **C-04**: **Preserve and transmit call transcript/context** to agents upon handoff, reducing customer repetition.

*(Document excerpt ends at C-04 / page 22 of 39 — Module C content beyond this point was not included in the provided source.)*
