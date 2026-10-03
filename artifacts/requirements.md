# AI-First Mobile Support Ticket Triage Platform — POC Requirements

**Document version:** 0.1  
**Status:** POC build specification  
**Primary consumer:** Coding agents and engineers  
**Last updated:** 2026-10-03

---

## 1. Introduction and Overview

### 1.1 Purpose

Build a proof-of-concept (POC) AI-first support ticket triage platform for a company operating an iOS and/or Android mobile application. End users submit support requests from the mobile application. The platform stores each ticket, enriches it with approved mobile/app context, asynchronously performs AI triage, and presents the structured triage result to support agents in a web console.

The POC must demonstrate that AI can reliably transform unstructured support requests into structured, actionable support tickets while keeping a human agent in control of consequential ticket decisions.

### 1.2 Business Goals

The POC must prove the platform can:

1. Reduce manual ticket triage effort and time.
2. Consistently classify common mobile-application support issues.
3. Recommend an appropriate priority and responsible support team.
4. Use mobile diagnostic context to improve triage quality.
5. Clearly distinguish user-provided facts, system metadata, and AI-generated inferences.
6. Allow support agents to review and override AI decisions.
7. Capture corrections as an evaluation dataset for future AI improvements.
8. Measure AI quality, latency, and cost rather than relying on subjective impressions.

### 1.3 POC Scope

The POC includes:

- A ticket-ingestion API suitable for an iOS/Android client.
- Storage of customer messages, attachments metadata, and approved diagnostic context.
- Asynchronous AI triage.
- Controlled ticket taxonomy and priority model.
- Structured AI output with confidence scores.
- A web-based support-agent queue.
- Ticket-detail and AI-review workflows.
- Human overrides and feedback capture.
- Basic administration of taxonomy, teams, and routing/priority rules.
- Basic operational and AI-quality analytics.

### 1.4 Out of Scope

The POC does **not** require:

- Multi-tenant SaaS architecture.
- Full CRM functionality.
- Voice, phone, WhatsApp, social-media, or email ingestion.
- Advanced workforce management.
- Advanced SLA management.
- Autonomous refunds, payments, account changes, or destructive actions.
- Autonomous ticket resolution without human review.
- Production-grade enterprise RBAC.
- Custom ML model training/fine-tuning.
- Large-scale knowledge-base RAG.
- Zendesk, Salesforce, ServiceNow, or similar integrations.
- Production-scale BI/reporting.
- Real-time chat between customer and agent.

### 1.5 Target Audience

- Mobile-application customers submitting support requests.
- Support agents reviewing and resolving requests.
- Support administrators configuring triage behavior.
- Engineering/product teams evaluating the POC.

### 1.6 Key Stakeholders and Roles

#### Customer

Uses the mobile application and submits a support request. The customer may provide text and optional attachments. The mobile application may attach explicitly approved diagnostic metadata.

#### Support Agent

Uses the web console to inspect incoming tickets, review AI triage results, modify classifications, assign tickets, change status, and resolve tickets.

#### Administrator

Configures categories/subcategories, teams, routing rules, priority rules, and AI-related configuration available in the POC.

#### AI Triage Service

Produces recommendations and structured analysis. It does not directly perform irreversible business actions.

---

## 2. Functional Requirements

Requirement IDs are normative and should be referenced by implementation tasks and tests.

### 2.1 Ticket Creation and Ingestion

**FR-TKT-001 — Create ticket**  
The system shall expose an authenticated API endpoint through which a mobile client can create a support ticket.

Minimum customer-supplied fields:

- `message`: required, non-empty string.
- `subject`: optional string.
- `attachments`: optional collection of attachment references/metadata.

**FR-TKT-002 — Generate ticket identity**  
The server shall generate an immutable unique ticket ID and creation timestamp for every successfully accepted ticket.

**FR-TKT-003 — Store before triage**  
The system shall persist the raw ticket before invoking AI triage. Failure or unavailability of the AI provider must not prevent ticket creation.

**FR-TKT-004 — Diagnostic context**  
The API shall accept optional mobile diagnostic context. Supported POC fields shall include, where available:

```json
{
  "platform": "ios | android",
  "app_version": "string",
  "os_version": "string",
  "device_model": "string",
  "locale": "string",
  "timezone": "string",
  "current_screen": "string",
  "account_tier": "string",
  "session_id": "string",
  "recent_error_codes": ["string"]
}
```

**FR-TKT-005 — Data provenance**  
The data model and UI shall distinguish at minimum:

- customer-provided content;
- client/system-provided diagnostic metadata;
- AI-generated analysis/inferences;
- human-agent decisions.

**FR-TKT-006 — Validation**  
The ingestion API shall reject malformed requests with machine-readable validation errors. Invalid diagnostic fields must not cause valid customer text to be silently discarded.

**FR-TKT-007 — Idempotency**  
The ticket-creation API shall support an idempotency key so mobile retries do not create duplicate tickets.

### 2.2 Ticket Lifecycle

**FR-STA-001 — Supported statuses**  
The POC shall support these ticket states:

- `NEW`
- `TRIAGING`
- `NEEDS_REVIEW`
- `OPEN`
- `IN_PROGRESS`
- `RESOLVED`
- `CLOSED`

**FR-STA-002 — Initial status**  
A newly persisted ticket shall enter `NEW`.

**FR-STA-003 — Triage transition**  
When asynchronous AI processing begins, the ticket shall transition to `TRIAGING`.

**FR-STA-004 — Successful triage**  
After successful triage, a ticket shall transition to `OPEN` unless configured review rules require `NEEDS_REVIEW`.

**FR-STA-005 — Review conditions**  
A ticket shall be eligible for `NEEDS_REVIEW` when one or more configured conditions are met, including low AI confidence, invalid/unrecognized classification, contradictory signals, or triage processing failure requiring human attention.

**FR-STA-006 — Transition audit**  
Every status change shall record old status, new status, actor/type, and timestamp.

### 2.3 Controlled Taxonomy

**FR-TAX-001 — Controlled values**  
AI classification shall use configured categories and subcategories. The AI must not persist invented taxonomy values as valid classifications.

**FR-TAX-002 — Initial taxonomy**  
Seed the POC with:

```text
ACCOUNT
  login
  registration
  verification
  profile
  account_deletion

BILLING
  payment_failed
  duplicate_charge
  refund
  subscription
  subscription_activation

TECHNICAL
  connectivity
  performance
  compatibility
  integration

BUG
  crash
  ui_issue
  incorrect_behavior
  data_issue
  notification_issue

FEATURE
  how_to
  feature_question
  feature_request
  unsupported_feature
```

**FR-TAX-003 — Administration**  
An administrator shall be able to activate/deactivate categories and subcategories without changing application source code.

### 2.4 AI Triage

**FR-AI-001 — Asynchronous execution**  
AI triage shall run asynchronously after ticket persistence.

**FR-AI-002 — Inputs**  
The triage service shall receive the customer message, subject when available, allowed diagnostic context, configured taxonomy, available teams, and applicable deterministic rules.

**FR-AI-003 — Structured output**  
The AI service shall return output conforming to a server-validated structured schema. At minimum:

```json
{
  "summary": "string",
  "category": "configured-category",
  "subcategory": "configured-subcategory",
  "priority": "P0 | P1 | P2 | P3",
  "routing": {
    "team": "configured-team"
  },
  "sentiment": "string",
  "tags": ["string"],
  "diagnosis": {
    "likely_cause": "string | null",
    "confidence": 0.0
  },
  "suggested_actions": ["string"],
  "suggested_reply": "string",
  "confidence": {
    "category": 0.0,
    "priority": 0.0,
    "routing": 0.0
  }
}
```

Confidence values shall be numeric values from `0.0` through `1.0`.

**FR-AI-004 — Schema validation**  
The server shall validate AI output before storing it as a successful triage result. Invalid output shall be retried according to the retry policy or routed to `NEEDS_REVIEW`.

**FR-AI-005 — Summary**  
The AI shall generate a concise summary describing the user's reported problem without adding unsupported facts.

**FR-AI-006 — Classification**  
The AI shall recommend one configured category and one valid subcategory belonging to that category.

**FR-AI-007 — Priority**  
The AI shall recommend one of `P0`, `P1`, `P2`, or `P3`.

**FR-AI-008 — Routing**  
The AI shall recommend a configured support team.

**FR-AI-009 — Tags**  
The AI may generate searchable tags. Tags must be normalized before persistence (trimmed, case-normalized, duplicate-free, and length-limited).

**FR-AI-010 — Sentiment**  
The AI may describe customer sentiment, but sentiment shall not by itself determine ticket priority.

**FR-AI-011 — Diagnosis**  
Likely-cause/diagnostic output shall be presented as an AI inference, not a verified fact. When evidence is insufficient, `likely_cause` may be `null`.

**FR-AI-012 — Suggested actions**  
The AI may recommend investigation steps for the support agent. Recommendations must not automatically execute account, payment, refund, deletion, or other consequential actions.

**FR-AI-013 — Suggested reply**  
The AI shall generate a draft response that an agent may review/edit. The POC shall not automatically send the draft to the customer.

**FR-AI-014 — Confidence**  
The system shall persist independent confidence scores for category, priority, and routing.

**FR-AI-015 — Prompt/model metadata**  
Each AI triage run shall record sufficient metadata for reproducibility/debugging, including provider/model identifier, prompt/configuration version, start/end timestamps, success/failure status, and token/cost data when exposed by the provider.

**FR-AI-016 — Preserve raw AI result**  
The platform shall retain the normalized triage result and may retain the raw provider result in a restricted diagnostic field/log, subject to data-retention policy.

### 2.5 Priority Rules

**FR-PRI-001 — Priority definitions**  
The initial priority model shall be:

- `P0 — Critical`: service unusable at broad scale, severe outage, or severe security-related issue requiring immediate review.
- `P1 — High`: core functionality unavailable for the affected customer, payment/subscription/access issue, or major user impact.
- `P2 — Normal`: individual functional issue, issue with workaround, or normal support request.
- `P3 — Low`: informational question, feature request, or cosmetic/non-blocking issue.

**FR-PRI-002 — Rules can override AI**  
The application shall support deterministic priority rules that can override an AI-recommended priority.

**FR-PRI-003 — Explain final source**  
The persisted final priority shall identify whether it originated from AI, deterministic rule override, or human override.

**FR-PRI-004 — Sentiment limitation**  
Negative sentiment alone shall not escalate a ticket to P0/P1.

### 2.6 Routing and Teams

**FR-RTE-001 — Configured teams**  
The system shall maintain a configurable collection of support teams with unique IDs and active/inactive status.

**FR-RTE-002 — Seed teams**  
The POC may seed teams such as `account_support`, `billing_support`, `technical_support`, and `product_support`.

**FR-RTE-003 — Final assignment**  
The platform shall persist both the AI-recommended team and the current/final assigned team so overrides are measurable.

### 2.7 Agent Queue

**FR-QUE-001 — Queue view**  
Authenticated support agents shall have a web view listing support tickets.

**FR-QUE-002 — Queue fields**  
Each queue row shall display at minimum ticket ID, concise summary/subject, priority, category, status, assigned/recommended team, AI confidence indicator, and creation time.

**FR-QUE-003 — Filters**  
Agents shall be able to filter by status, priority, category, team, and review-needed state.

**FR-QUE-004 — Search**  
Agents shall be able to search tickets by ticket ID and customer-supplied textual content. Search of other indexed fields is optional for the POC.

**FR-QUE-005 — Sorting**  
Agents shall be able to sort by creation time and priority.

### 2.8 Ticket Detail and Review

**FR-DET-001 — Original content**  
The detail view shall prominently show the original customer message separately from AI-generated content.

**FR-DET-002 — Context**  
The detail view shall show available mobile diagnostic context with clear provenance.

**FR-DET-003 — AI analysis**  
The detail view shall show summary, classification, priority, routing, tags, sentiment, diagnosis, suggested actions, suggested reply, and confidence values when available.

**FR-DET-004 — Agent approval**  
An agent shall be able to accept the AI triage result.

**FR-DET-005 — Agent correction**  
An agent shall be able to change category, subcategory, priority, assigned team, and tags.

**FR-DET-006 — Agent response editing**  
An agent shall be able to edit the suggested reply. Sending the response externally is out of scope unless a later integration is added.

**FR-DET-007 — Status management**  
An agent shall be able to transition a ticket through allowed lifecycle states.

### 2.9 Feedback and Evaluation Dataset

**FR-FBK-001 — Preserve prediction**  
Human edits must not overwrite the original AI prediction. Both values shall remain available.

**FR-FBK-002 — Record correction**  
For each corrected triage field, store the AI value, human value, agent identifier, and timestamp.

**FR-FBK-003 — Approval signal**  
Accepting an unchanged AI decision shall be recordable as an approval signal.

**FR-FBK-004 — Evaluation export/query**  
The system shall provide a query/API or export suitable for comparing AI predictions with final human decisions.

### 2.10 Administration

**FR-ADM-001 — Taxonomy management**  
Administrators shall be able to view, create, edit, activate, and deactivate taxonomy entries, subject to referential-integrity constraints.

**FR-ADM-002 — Team management**  
Administrators shall be able to view, create, edit, activate, and deactivate support teams.

**FR-ADM-003 — Rule management**  
Administrators shall be able to configure basic deterministic priority/routing rules without modifying application source code.

**FR-ADM-004 — AI thresholds**  
Administrators shall be able to configure at least the low-confidence threshold used to send tickets to `NEEDS_REVIEW`.

### 2.11 Analytics

**FR-ANL-001 — POC metrics**  
The platform shall calculate or expose data sufficient to calculate:

- category agreement/accuracy against reviewed tickets;
- subcategory agreement/accuracy;
- priority agreement;
- routing agreement;
- human override rate;
- low-confidence/review rate;
- AI triage latency;
- triage failure rate;
- estimated AI cost per ticket when provider cost/token information is available.

**FR-ANL-002 — Denominator clarity**  
Accuracy/agreement metrics shall distinguish reviewed tickets from unreviewed tickets. Unreviewed AI predictions shall not be treated as confirmed-correct labels.

### 2.12 Audit Trail

**FR-AUD-001 — Audited actions**  
The system shall record audit events for ticket creation, AI triage completion/failure, classification changes, priority changes, team changes, status changes, and agent approval/correction.

**FR-AUD-002 — Audit fields**  
Audit events shall include ticket ID, event type, actor or actor type, timestamp, and relevant before/after values.

---

## 3. Non-Functional Requirements

### 3.1 Performance

**NFR-PERF-001 — Ingestion latency**  
Under normal POC load, the ticket-creation API shall acknowledge a successfully persisted ticket within 1 second at the 95th percentile, excluding attachment upload time.

**NFR-PERF-002 — AI latency target**  
The target end-to-end AI triage time shall be <= 10 seconds at the 95th percentile under normal POC load. This is a target rather than a guarantee because external model-provider latency may vary.

**NFR-PERF-003 — UI response**  
Common queue/filter/detail API requests should complete within 1 second at the 95th percentile for the POC dataset.

**NFR-PERF-004 — POC capacity**  
The implementation shall support at least 10,000 stored tickets and bursts of at least 10 new ticket submissions per second without data loss. AI work may queue during bursts.

### 3.2 Reliability and Failure Handling

**NFR-REL-001 — Durable ingestion**  
Once the API acknowledges successful ticket creation, the ticket must remain stored even if downstream AI processing fails.

**NFR-REL-002 — Retry**  
Transient AI/provider failures shall use bounded retry with backoff. Retry count and terminal failure status shall be configurable.

**NFR-REL-003 — No infinite processing**  
A triage job must not remain indefinitely in `TRIAGING`; terminal failure shall result in a reviewable state and error record.

**NFR-REL-004 — Idempotent workers**  
Triage processing shall be safe to retry without creating duplicate active triage results or corrupting ticket state.

### 3.3 Availability

**NFR-AVL-001 — POC availability target**  
Target service availability is 99% during active POC evaluation periods, excluding planned maintenance and external-provider outages.

**NFR-AVL-002 — AI degradation**  
An AI-provider outage shall degrade triage functionality but shall not make ticket ingestion unavailable.

### 3.4 Security

**NFR-SEC-001 — Transport security**  
All client/server and server/third-party communication shall use TLS/HTTPS in deployed environments.

**NFR-SEC-002 — Authentication**  
Mobile ingestion and agent/admin web APIs shall require authentication appropriate to the client type.

**NFR-SEC-003 — Authorization**  
At minimum, the system shall distinguish agent and administrator permissions for administrative operations.

**NFR-SEC-004 — Secret management**  
API keys, model-provider credentials, database passwords, and signing secrets shall be loaded from environment/secret management and shall never be committed to source control.

**NFR-SEC-005 — Encryption at rest**  
Production-like POC deployments shall use storage/database encryption at rest where supported by the hosting platform.

**NFR-SEC-006 — Logging hygiene**  
Authentication secrets, tokens, raw credentials, and unnecessary PII shall not be written to application logs.

**NFR-SEC-007 — Input safety**  
All user-provided text shall be treated as untrusted input. The system shall use normal web security controls against injection/XSS and shall not treat instructions contained inside a support ticket as privileged system instructions.

### 3.5 Privacy and Data Handling

**NFR-PRV-001 — Data minimization**  
Only diagnostic fields required by the POC shall be collected. The mobile client shall not upload arbitrary device data by default.

**NFR-PRV-002 — Diagnostic consent/notice**  
The mobile support experience shall clearly indicate when diagnostic information is included and allow the product to implement an opt-in/opt-out mechanism appropriate to its policy.

**NFR-PRV-003 — Retention configurability**  
Ticket data, AI diagnostic records, and audit data shall have configurable retention periods, even if the POC initially uses one default value.

**NFR-PRV-004 — Provider minimization**  
Only fields needed for triage shall be sent to the AI provider.

### 3.6 Observability

**NFR-OBS-001 — Structured logs**  
Backend services shall emit structured logs with correlation identifiers such as ticket ID and triage-job ID where applicable.

**NFR-OBS-002 — Metrics**  
The system shall expose operational metrics for API requests, error rates, triage jobs, AI latency, retries, and failures.

**NFR-OBS-003 — Traceability**  
An operator shall be able to trace a ticket from ingestion through AI triage to the stored result and human review using identifiers rather than message contents.

### 3.7 Maintainability

**NFR-MNT-001 — Configuration over hard-coding**  
Taxonomy, teams, confidence thresholds, model identifier, and basic triage rules shall be configurable.

**NFR-MNT-002 — AI provider abstraction**  
Application business logic shall access AI triage through a defined service/interface rather than embedding provider-specific calls throughout the codebase.

**NFR-MNT-003 — Schema migrations**  
Database schema changes shall be managed through versioned migrations.

**NFR-MNT-004 — Automated tests**  
Core domain logic, API validation, state transitions, deterministic rules, AI-output validation, and override preservation shall have automated tests.

---

## 4. External Interface Requirements

### 4.1 Mobile Application Interface

The POC shall expose a REST/JSON interface for mobile ticket creation. A suggested contract is:

```http
POST /api/v1/tickets
Idempotency-Key: <client-generated-key>
Authorization: Bearer <token>
Content-Type: application/json
```

Example request:

```json
{
  "subject": "Premium not activated",
  "message": "I paid for premium but the app still says I am on the free plan.",
  "diagnostics": {
    "platform": "ios",
    "app_version": "2.4.1",
    "os_version": "26.1",
    "device_model": "iPhone",
    "locale": "en-SA",
    "timezone": "Asia/Riyadh",
    "current_screen": "subscription",
    "account_tier": "premium",
    "session_id": "session-123",
    "recent_error_codes": ["SUBSCRIPTION_SYNC_FAILED"]
  }
}
```

Example response:

```json
{
  "ticket_id": "T-1042",
  "status": "NEW",
  "created_at": "2026-10-03T12:00:00Z"
}
```

Exact route naming may vary, but semantics must satisfy the requirements above.

### 4.2 Agent/Admin Web Interface

The web client shall consume authenticated backend APIs for:

- ticket queue/list;
- search/filter/sort;
- ticket detail;
- ticket updates;
- AI review/approval;
- human corrections;
- status transitions;
- taxonomy configuration;
- team configuration;
- rule/threshold configuration;
- basic analytics.

The backend, not the browser, shall be authoritative for validation and permissions.

### 4.3 AI Provider Interface

The system requires one LLM provider capable of structured output or JSON-schema-compatible responses.

The integration shall:

- be server-side only;
- support configurable provider/model identifiers;
- enforce a strict output schema at the application boundary;
- use timeouts;
- implement bounded retries;
- record latency and errors;
- record token/cost metadata when provided;
- avoid granting the model direct database mutation or consequential external-action permissions.

No specific vendor is mandated by this requirements document.

### 4.4 Database

A transactional relational database is recommended for the POC. The implementation must support entities equivalent to:

- users/agents/admins;
- tickets;
- ticket diagnostics;
- attachments metadata;
- taxonomy categories/subcategories;
- support teams;
- triage jobs/runs;
- AI triage predictions;
- final/human triage decisions;
- feedback/corrections;
- rules/configuration;
- audit events.

The exact schema is an implementation decision as long as original AI predictions and subsequent human decisions remain independently recoverable.

### 4.5 Asynchronous Job Interface

AI triage shall execute through an asynchronous job mechanism/queue. The implementation may use a dedicated message broker, managed queue, or database-backed job queue appropriate to the POC.

The queue/job layer must support:

- enqueue after ticket persistence;
- retry;
- terminal failure;
- idempotent processing;
- job status visibility;
- correlation to ticket ID.

### 4.6 Attachment Storage

If screenshot/file upload is implemented, binary attachment content shall be stored in object/blob storage rather than directly in relational rows. The database shall retain metadata and storage references.

POC attachment constraints:

- screenshots/images are sufficient;
- file type and size must be validated;
- object access must not be public by default;
- malware scanning is recommended for a production version but is not required for initial POC acceptance unless arbitrary file types are enabled.

### 4.7 Authentication Interface

The POC may use a managed identity provider or locally implemented development authentication. Regardless of implementation:

- mobile requests must have a verifiable application/user identity or POC test identity;
- agent/admin sessions must be authenticated;
- admin-only endpoints must reject agent-only credentials.

---

## 5. Assumptions, Constraints, and Acceptance Criteria

### 5.1 Assumptions

1. The POC serves one mobile product/organization rather than multiple external tenants.
2. The initial support language is English. Multilingual support may be added later.
3. Mobile clients can provide the defined diagnostic metadata where platform permissions allow.
4. AI recommendations are advisory and remain reviewable by humans.
5. A third-party LLM API is available during normal POC operation.
6. The platform has a small number of support teams and a controlled taxonomy.
7. POC analytics may be implemented as simple dashboard values or backend queries rather than a full BI system.
8. Customer reply delivery is outside the POC; `suggested_reply` is a draft for agent use.

### 5.2 Constraints

1. Ticket creation must not synchronously depend on AI completion.
2. The AI must not invent and persist new categories, subcategories, teams, or priority values.
3. Human corrections must never destroy the original AI prediction.
4. AI-generated diagnoses must be visually/data-model distinguishable from verified customer/system facts.
5. Sentiment alone cannot determine critical/high priority.
6. No AI output may directly trigger refunds, payments, account deletion, access changes, or similar consequential actions in the POC.
7. The system must remain usable for ticket ingestion when the AI provider is unavailable.
8. Provider credentials must never be exposed to the mobile or browser clients.
9. Implementation technology choices are intentionally not prescribed unless needed to satisfy a requirement.

### 5.3 Definition of Done for the POC

The POC is complete when the following end-to-end scenario works in a deployed/test environment:

1. A mobile/test client submits a support message plus optional diagnostic context.
2. The backend persists the ticket and immediately returns its ID without waiting for AI completion.
3. An asynchronous worker processes the ticket through the configured AI provider.
4. The AI output is schema-validated and stored.
5. The ticket appears in the agent queue with category, priority, routing, confidence, and status.
6. An agent opens the ticket and can separately inspect original content, diagnostics, and AI inferences.
7. The agent can approve the triage result or change category, subcategory, priority, team, and tags.
8. The system preserves the AI prediction and records the human decision/correction.
9. The ticket can move through the defined lifecycle to `RESOLVED` and `CLOSED`.
10. Basic metrics can compare reviewed AI predictions with human-final decisions.
11. If the AI provider is disabled/unavailable, new tickets are still created and remain recoverable/reviewable.

### 5.4 Objective Acceptance Criteria

#### AC-001 — Ticket ingestion

**Given** a valid authenticated ticket request,  
**when** the mobile client submits it,  
**then** the API returns a unique ticket ID and the persisted ticket can be retrieved.

#### AC-002 — Idempotent creation

**Given** the same authenticated request and idempotency key,  
**when** the client retries ticket creation,  
**then** no duplicate ticket is created.

#### AC-003 — AI-independent ingestion

**Given** the AI provider is unavailable,  
**when** a valid ticket is submitted,  
**then** the ticket is persisted successfully and is visible for later processing/review.

#### AC-004 — Structured triage

**Given** a stored ticket and functioning AI provider,  
**when** triage completes,  
**then** the stored result contains a valid configured category/subcategory, valid priority, valid team, summary, confidence scores, and other schema-defined fields.

#### AC-005 — Invalid AI output

**Given** an AI response that violates the schema or references invalid controlled values,  
**when** the backend validates it,  
**then** it is not accepted as a successful final triage result and the retry/failure workflow is executed.

#### AC-006 — Low-confidence review

**Given** a triage result below the configured confidence threshold,  
**when** processing completes,  
**then** the ticket is marked/routed for human review according to configuration.

#### AC-007 — Human override preservation

**Given** an AI category of `TECHNICAL` and an agent changing it to `BILLING`,  
**when** the correction is saved,  
**then** both `TECHNICAL` (AI prediction) and `BILLING` (human/final decision) remain queryable with correction metadata.

#### AC-008 — Deterministic priority override

**Given** a configured deterministic rule that overrides an AI priority,  
**when** the rule matches,  
**then** the final priority reflects the rule and its source is recorded as a rule override.

#### AC-009 — Queue operation

**Given** multiple tickets,  
**when** an agent filters by priority, status, category, or team,  
**then** only matching tickets are returned/displayed.

#### AC-010 — Lifecycle audit

**Given** a ticket that changes status,  
**when** each transition occurs,  
**then** an audit record contains the previous state, new state, actor/type, and timestamp.

#### AC-011 — AI provenance

**Given** a ticket with an AI-generated likely cause,  
**when** an agent views the detail page,  
**then** the likely cause is clearly presented as AI-generated/inferred and not as customer-provided fact.

#### AC-012 — Security boundaries

**Given** an unauthenticated request to protected agent/admin APIs,  
**then** access is denied; and **given** an authenticated non-admin agent requesting an admin-only mutation, **then** access is denied.

#### AC-013 — Secrets

Automated repository/configuration checks shall find no committed live provider API keys, database credentials, or signing secrets.

#### AC-014 — Performance

Under the agreed POC load test, ticket ingestion meets the p95 <= 1 second target excluding attachment upload, and no acknowledged tickets are lost.

#### AC-015 — Evaluation metrics

**Given** a set of agent-reviewed tickets,  
**when** evaluation metrics are requested,  
**then** the system can report category, priority, and routing agreement plus override rate using only reviewed tickets as ground truth.

### 5.5 Recommended Build Order

This order is informative rather than normative:

1. Domain model, database migrations, and seed taxonomy/teams.
2. Ticket-ingestion API and idempotency.
3. Ticket lifecycle and audit events.
4. Asynchronous job processing.
5. AI-provider abstraction and structured triage schema.
6. AI triage persistence, retries, and review thresholds.
7. Agent queue and ticket-detail UI.
8. Agent approval/correction workflow and feedback persistence.
9. Admin configuration.
10. Metrics/analytics and end-to-end acceptance tests.

---

## Appendix A — Canonical POC Flow

```text
Mobile App
    |
    | POST ticket + approved diagnostics
    v
Support API
    |
    +--> Persist ticket (NEW)
    |
    +--> Enqueue triage job
              |
              v
         AI Triage Worker
              |
              +--> TRIAGING
              +--> LLM structured analysis
              +--> Validate output
              +--> Apply deterministic rules
              |
              v
        OPEN / NEEDS_REVIEW
              |
              v
         Agent Web Console
              |
              +--> Approve AI decision
              |        or
              +--> Correct AI decision
              |
              v
        Feedback + Audit Data
              |
              v
       Evaluation Metrics
```

## Appendix B — Core Design Principle

**The AI proposes structured decisions; the application validates and governs those decisions; a human can review and override them.**

This principle should guide implementation choices whenever behavior is not explicitly specified elsewhere in this document.
