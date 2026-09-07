# BRIEF.md — Technology Choices & Rationale

Scope of reasoning: only the pieces that are still open (backend framework, AI layer,
supporting infra). Anything CBC already mandates (Backbase, OAuth2/OIDC, OpenAPI,
WCAG 2.2 AA) is treated as fixed and isn't re-justified here.

---

## 1. Backend Framework — FastAPI (over Ruby)

| Requirement driving the decision | FastAPI | Ruby (Rails/Sinatra) |
|---|---|---|
| OpenAPI is the mandated contract-first standard (PROBLEMS #2) | Generates OpenAPI 3.1 directly from Pydantic models — the spec and the code can't drift apart | Needs a separate gem (rswag/committee) bolted on; spec and code are two artifacts to keep in sync |
| UI Intent Schema versioning (PROBLEMS #3) | Pydantic discriminated unions map cleanly onto a card/form/action schema with compile-time validation | Doable, but weaker static typing makes schema-version drift harder to catch before runtime |
| Streaming chat responses (FEATURES #48/#23) | Native `async`/`StreamingResponse`, first-class async SDKs for Bedrock streaming (`invoke_model_with_response_stream`) | Requires ActionCable or SSE workarounds; AWS SDK for Ruby has weaker async story for streamed model output |
| Bedrock/AWS integration | `boto3` is the reference AWS SDK — Bedrock, Bedrock Agents, Knowledge Bases, Guardrails APIs land there first | `aws-sdk-bedrock` (Ruby) exists but trails Python SDK in coverage and examples for newer Bedrock features |
| Shared core between Coach (app) and Voice AI (Module C) | Same async service can host both a REST/WebSocket API and a voice-turn handler behind the same orchestrator class — avoids duplicating the NLU/state-machine logic (PLAN.md workstream 4) | Same is possible, but the ecosystem for real-time STT/TTS integration (voice SDKs, telephony webhooks) is more mature on the Python side for this use case |

**Decision: FastAPI.** The deciding factors are specific to this project, not general framework preference: contract-first OpenAPI is a hard requirement, and Bedrock's newest capabilities (Guardrails, Knowledge Bases, Agents, streaming) ship in `boto3` first.

---

## 2. AI Layer — Amazon Bedrock, mapped to specific problems

Bedrock is used where it maps to a concrete open problem, not wholesale:

- **Intent/entity extraction + Taglish handling** (PROBLEMS #8, #11): a Bedrock-hosted Claude model does intent + entity extraction; language identification (BCP 47 tagging) runs as a pre-classification step so code-switched input is tagged before routing to locale content.
- **RAG for Insights/"Why"/Financial Coaching** (PROBLEMS #12): **Bedrock Knowledge Bases** handles ingestion from ECMS + vectorization + retrieval, so the team isn't hand-building an embedding/chunking/retrieval pipeline — this directly removes one whole workstream item.
- **No-unsafe-claims grounding layer** (PROBLEMS #13): **Bedrock Guardrails** sits between the model and the UI as the policy-enforcement point the RFP explicitly asks for (not just a disclaimer) — denied topics, PII redaction, and grounding checks against retrieved content are configured here rather than hand-rolled.
- **Explainability / basis logging** (PROBLEMS #14, B-06, B-AT-04): every Bedrock invocation carries a structured citation output (from the Knowledge Base retrieval step) that's persisted as the "basis" object — the audit requirement is satisfied by logging what the RAG layer already returns, not a bolt-on afterward.
- **Voice AI (Module C)**: **Amazon Transcribe** (STT) and **Amazon Polly** (TTS) are the natural pairing here since both have Philippine English/Filipino language support and sit in the same AWS account/VPC as the rest of the stack — avoids a third-party STT/TTS vendor and a second set of data-residency/compliance reviews.
- **Model-risk monitoring** (PROBLEMS #37, Optimization phase): Bedrock's per-invocation logging (CloudWatch/CloudTrail) is the raw feed for the drift dashboards required in the Optimization phase — this is set up once in Foundation, not retrofitted, per the WBS sequencing in PLAN.md.

---

## 3. Supporting Stack

| Layer | Choice | Why (tied to this project) |
|---|---|---|
| Session/journey state (PROBLEMS #9, B-AT-03) | Redis (Amazon ElastiCache) | Sub-ms reads for "resume where you left off" across tabs/backgrounding; TTL support fits the resume-token expiry requirement in Flow 2 |
| Vector store for RAG | Bedrock Knowledge Bases (backed by OpenSearch Serverless) | Keeps ingestion, embedding, and retrieval inside the managed Bedrock pipeline instead of standing up/maintaining a separate vector DB |
| Async job/event handling (PROBLEMS #6, #77) | Amazon EventBridge + SQS | Matches the "event bus for async flows" requirement directly; EventBridge's schema registry doubles as the common event envelope the RFP asks for |
| Identity federation (PROBLEMS #16, #20) | Amazon Cognito (or direct OIDC to CBC's IdP) fronting Entra ID for CRM scopes | Cognito brokers OIDC cleanly for the mobile app session; Entra ID/MSAL is used specifically for the Dataverse/D365 leg since that's CBC's existing CRM identity path — not replacing it |
| Idempotent transaction execution (PROBLEMS #19) | DynamoDB conditional writes keyed on `Idempotency-Key` | Sits directly in front of the Action Executor; a single-digit-ms conditional check is enough to reject a duplicate submit from a retried mobile request |
| Compute | AWS Fargate (ECS) for the orchestration service | Matches the async/streaming FastAPI workload without managing servers; scales per-persona traffic patterns (retail bursts vs. steady contact-center voice load) independently if split into two services later |
| Audit/compliance store (PROBLEMS #34, #38) | Aurora PostgreSQL (append-only audit tables) + S3 for long-term retention | Relational store gives queryable consent/audit records for dispute handling; S3 lifecycle rules handle the retention/breach-notification timeline workflow |
| Secrets/keys | AWS Secrets Manager + KMS | Needed regardless of AI choice, but called out because step-up auth tokens, resume tokens, and Idempotency-Key signing all need managed key rotation, not hardcoded secrets |

---

## 4. What's Still Open
- Contact-center integration library for Genesys (native AppFoundry connector vs. custom telephony webhook bridge) — depends on CBC's Genesys deployment specifics, not yet confirmed.
- Whether Voice AI (Module C) runs as a second FastAPI service sharing the orchestrator core, or a separate deployable — recommended: same core, separate deployment unit, per PLAN.md Section 6.
