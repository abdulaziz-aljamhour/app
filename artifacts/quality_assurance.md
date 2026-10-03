# AI-First Mobile Support Ticket Triage Platform — Quality Assurance

**Document version:** 0.1  
**Status:** POC QA specification  
**Primary consumer:** Coding agents, engineers, and evaluators  
**Last updated:** 2026-10-03  
**Source requirements:** [requirements.md](./requirements.md) §5.3–5.4, NFR-MNT-004  
**Related artifacts:** [system_design.md](./system_design.md) §11, [api_contracts.md](./api_contracts.md), [data_model.md](./data_model.md), [ui_ux_guidelines.md](./ui_ux_guidelines.md)

---

## 1. Purpose and Scope

This document defines how the POC is verified: test strategy, suites, environments, AI-quality evaluation, security/privacy checks, UX gates, defect severity, and release exit criteria.

QA exists to prove the platform can turn unstructured mobile support requests into structured, reviewable tickets **without** treating AI as authority or losing human-final decisions.

### 1.1 Normative QA principle

> **The AI proposes structured decisions; the application validates and governs those decisions; a human can review and override them.**

Every test suite must protect at least one of: ingestion durability, schema governance, provenance separation, human override integrity, or safe degradation when AI is unavailable.

### 1.2 In scope

- Automated unit, API/integration, worker, and contract tests
- End-to-end acceptance (AC-001–015) and Definition of Done (§5.3 in requirements)
- AI output validation, retry/failure, confidence gating, and evaluation metrics
- Agent/admin UI provenance and workflow checks
- Performance smoke, reliability failure-mode drills, security/privacy hygiene
- Seed data, fixtures, stubs, and CI gates for POC completion

### 1.3 Out of scope

- Production SRE runbooks, chaos engineering at scale, or 24/7 on-call
- Formal model fine-tuning evaluation pipelines
- Mobile customer UI QA (owned by the mobile app; API contract only)
- Penetration testing beyond POC boundary checks (AC-012/013, NFR-SEC-*)
- External CRM / omnichannel / real-time chat verification

---

## 2. Quality Objectives

| Objective | Success signal | Primary refs |
|---|---|---|
| Durable ingestion | Create returns ID; ticket survives AI/Redis failure | AC-001, AC-003, NFR-REL-001, NFR-AVL-002 |
| Governed AI | Invalid/invented values never become successful finals | AC-004, AC-005, FR-AI-004, FR-TAX-001 |
| Human control | Overrides preserve AI prediction + correction metadata | AC-007, FR-FBK-001–002 |
| Provenance clarity | UI/API never present AI as customer fact | AC-011, FR-TKT-005, UI §1.1 |
| Measurable quality | Reviewed-only agreement/override metrics | AC-015, FR-ANL-002 |
| Safe operations | Authz boundaries, no committed secrets, prompt safety | AC-012, AC-013, NFR-SEC-* |
| Usable under load | Ingestion p95 ≤ 1s; no lost acks at POC burst | AC-014, NFR-PERF-001/004 |

---

## 3. Test Strategy

### 3.1 Pyramid (normative for automation)

| Layer | Tooling | Owns |
|---|---|---|
| Unit | `pytest` | State machine, tag normalization, rules engine, AI schema validation, confidence gating, sentiment≠priority, request hashing |
| Integration / API | `pytest` + `httpx` + test DB | REST contracts, idempotency, authz, queue filters, audit writes, analytics denominators |
| Worker | `pytest` + Celery eager/fake Redis + stub LLM | Triage job lifecycle, retries, dead-letter, immutable AI persist, rule application |
| Contract | Pydantic schema fixtures / OpenAPI-aligned snapshots | Mobile create body, agent detail provenance shape, AI structured output |
| UI (lightweight) | HTMX response assertions + optional Playwright smoke | Provenance labels (AC-011), legal status transitions, review badge |
| E2E / acceptance | Compose stack + scripted scenarios | Requirements §5.3 flow; AC suite |
| Perf smoke | Locust or k6 | AC-014 / NFR-PERF-001 create path |
| Static hygiene | gitleaks / CI grep | AC-013 secrets |

### 3.2 Stub vs live AI policy

| Mode | When | Rules |
|---|---|---|
| **Stub LLM (default CI)** | PR and merge gates | Deterministic JSON fixtures; cover valid, invalid schema, invented taxonomy, low confidence |
| **Live LLM (optional nightly / eval)** | Staging with real key | Sample golden tickets; record latency/cost; never required for merge if stubs cover AC-004/005/006 |

Live calls must not be the only path that proves schema validation or retry behavior.

### 3.3 Environments

| Env | Purpose | Dependencies |
|---|---|---|
| Local unit | Fast feedback | SQLite/Postgres test container optional; no MinIO required for most units |
| Local/CI integration | AC automation | PostgreSQL, Redis (or fakes), MinIO optional, stub LLM |
| Compose POC | E2E DoD + demos | Full stack per [system_design.md](./system_design.md) §10 |
| Staging (optional) | Live AI + TLS check | Real provider; secrets from env only |

### 3.4 Fixtures and seed data

Minimum seed for QA (align with FR-TAX-002, FR-RTE-002):

- Taxonomy categories/subcategories (active + one inactive for negative tests)
- Support teams (`billing_support`, etc.)
- Users: `mobile`, `agent`, `admin` JWTs or local credentials
- At least one deterministic priority rule (AC-008)
- Configurable `LOW_CONFIDENCE_THRESHOLD`
- Factory helpers: ticket + diagnostics + triage job + AI result + decision + correction

Golden ticket scenarios (informative):

1. Billing / subscription activation (diagnostics with error codes)
2. Technical crash with screenshot metadata
3. Account access / login
4. Low-confidence / ambiguous message → `NEEDS_REVIEW`
5. Rule-forced P0/P1 override despite milder AI priority
6. Sentiment “angry” alone must not force critical priority (FR-PRI-004)

---

## 4. Traceability Matrix (AC-001–015)

| ID | Title | Automated primary | Manual / UI supplement | Blocker if fail? |
|---|---|---|---|---|
| AC-001 | Ticket ingestion | API: POST → GET | — | Yes |
| AC-002 | Idempotent creation | Double POST same key; body-hash conflict → 409 | — | Yes |
| AC-003 | AI-independent ingestion | LLM/Redis down; create 201 + visible | Compose kill worker/LLM | Yes |
| AC-004 | Structured triage | Stub LLM → validated stored fields | Spot-check live model | Yes |
| AC-005 | Invalid AI output | Bad schema / invented codes → not success final | — | Yes |
| AC-006 | Low-confidence review | Below threshold → `NEEDS_REVIEW` | Queue review badge | Yes |
| AC-007 | Human override preservation | Correct category; AI + human both queryable | Detail panels | Yes |
| AC-008 | Deterministic priority override | Seed rule → `priority_source=rule` | Decision panel label | Yes |
| AC-009 | Queue operation | Filter assertions | Console filter smoke | Yes |
| AC-010 | Lifecycle audit | Transition → audit fields | Audit snippet UI | Yes |
| AC-011 | AI provenance | Template/API `provenance: ai` labeling | Visual review checklist | Yes |
| AC-012 | Security boundaries | 401 / 403 matrix | Login smoke | Yes |
| AC-013 | Secrets | CI secret scan | — | Yes |
| AC-014 | Performance | Locust/k6 create path | — | Soft for early MVP; hard for DoD |
| AC-015 | Evaluation metrics | Reviewed-only denominators | Analytics page copy | Yes |

Mapping to design tests: [system_design.md](./system_design.md) §11.

---

## 5. Functional Test Suites

IDs below are QA case IDs (`QA-*`). Link FR/AC IDs in test names/docstrings.

### 5.1 Ingestion (`QA-TKT-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-TKT-01 | Valid create | `201`, `ticket_id`, status `NEW`, durable row (AC-001) |
| QA-TKT-02 | Missing message | `422` validation error shape |
| QA-TKT-03 | Diagnostics allowlist | Unknown keys rejected/ignored without dropping message |
| QA-TKT-04 | Idempotent replay | Same principal + key → same ticket, no duplicate (AC-002) |
| QA-TKT-05 | Idempotency conflict | Same key, different body hash → `409` |
| QA-TKT-06 | Unauthenticated create | `401` |
| QA-TKT-07 | Agent role cannot create (strict mode) | `403` if role matrix enforced |
| QA-TKT-08 | Attachment MIME/size | Reject non-image / oversize; metadata-only in DB |
| QA-TKT-09 | Create without LLM | Provider/Redis unavailable → still persist (AC-003) |

### 5.2 Lifecycle and audit (`QA-STA-*`, `QA-AUD-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-STA-01 | Initial status | Create → `NEW` |
| QA-STA-02 | Triage start | Job run → `TRIAGING` |
| QA-STA-03 | Success path | Valid AI → `OPEN` or `NEEDS_REVIEW` per rules |
| QA-STA-04 | Illegal transition | `409` / rejected; UI hides illegal options |
| QA-STA-05 | Resolve/close | Allowed transitions with confirmation UX |
| QA-AUD-01 | Status change audit | previous, new, actor, timestamp (AC-010) |
| QA-AUD-02 | Decision/correction audit | Consequential edits recorded (FR-AUD-*) |

### 5.3 AI triage worker (`QA-AI-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-AI-01 | Happy path structured output | All required fields + valid controlled values (AC-004) |
| QA-AI-02 | Schema violation | Not accepted as success; retry then reviewable failure (AC-005) |
| QA-AI-03 | Invented category/team/priority | Rejected as invalid |
| QA-AI-04 | Inactive taxonomy code | Rejected |
| QA-AI-05 | Low confidence | Persist AI; status `NEEDS_REVIEW` (AC-006) |
| QA-AI-06 | Bounded retries | Transient 5xx → backoff; no infinite loop (NFR-REL-002/003) |
| QA-AI-07 | Worker idempotency | Redelivery → one active AI result / no corrupt state (NFR-REL-004) |
| QA-AI-08 | Stuck TRIAGING reaper | Past timeout → reviewable terminal state |
| QA-AI-09 | Prompt metadata stored | model, prompt_version, latency, cost fields (FR-AI-015) |
| QA-AI-10 | Raw result preserved | Restricted storage; AI row immutable after success (FR-AI-016, FR-FBK-001) |
| QA-AI-11 | Provider payload minimization | Only allowlisted fields sent to LLM (NFR-PRV-004) |
| QA-AI-12 | Ticket text untrusted | Instruction-like customer text does not elevate privileges (NFR-SEC-007) |

### 5.4 Rules and priority (`QA-PRI-*`, `QA-RTE-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-PRI-01 | Rule overrides AI priority | Final priority from rule; `priority_source=rule` (AC-008) |
| QA-PRI-02 | Source explanation | Decision exposes AI vs rule vs human (FR-PRI-003) |
| QA-PRI-03 | Sentiment alone | Angry/frustrated cannot alone force P0/P1 (FR-PRI-004) |
| QA-RTE-01 | Team assignment | Final team from AI/rule/human per config (FR-RTE-003) |

### 5.5 Agent queue and detail (`QA-QUE-*`, `QA-DET-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-QUE-01 | Filters | status, priority, category, team, needs_review (AC-009) |
| QA-QUE-02 | Search | ticket id / text match |
| QA-QUE-03 | Sort | created_at / priority |
| QA-DET-01 | Provenance panels | customer / diagnostics / ai / decision separated (AC-011) |
| QA-DET-02 | Approve | Final = AI; approval signal recorded (FR-FBK-003) |
| QA-DET-03 | Correct category | AI remains; human decision + correction row (AC-007) |
| QA-DET-04 | Edit suggested reply | Draft editable; no auto-send |
| QA-DET-05 | Confidence display | Per-field category/priority/routing visible |

### 5.6 Admin (`QA-ADM-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-ADM-01 | Taxonomy CRUD | Agent forbidden; admin allowed (AC-012) |
| QA-ADM-02 | Teams / rules / threshold | Config affects triage without redeploy (NFR-MNT-001) |
| QA-ADM-03 | Referential integrity | Deactivate-in-use values blocked or explained |

### 5.7 Analytics and feedback (`QA-ANL-*`, `QA-FBK-*`)

| Case | Intent | Expect |
|---|---|---|
| QA-FBK-01 | Correction fields | before/after + actor + timestamp |
| QA-ANL-01 | Reviewed-only metrics | Unreviewed AI not counted as ground truth (AC-015, FR-ANL-002) |
| QA-ANL-02 | Agreement rates | category, priority, routing + override rate |
| QA-ANL-03 | Ops metrics exposure | Prometheus counters/histograms present (NFR-OBS-002) |

---

## 6. Non-Functional Verification

### 6.1 Performance

| Check | Target | Method |
|---|---|---|
| Ingestion p95 | ≤ 1s excluding attachment upload | Locust/k6 against create (AC-014, NFR-PERF-001) |
| AI triage p95 | ≤ 10s target (soft) | Worker metrics / stub+delay scenarios (NFR-PERF-002) |
| Queue/detail p95 | ≤ 1s on POC dataset | API timing under seeded 10k tickets (NFR-PERF-003) |
| Capacity / burst | ≥ 10k tickets; ≥ 10 creates/s no loss | Burst create; assert ack count = row count (NFR-PERF-004) |

Pass rule for DoD: AC-014 hard; AI latency soft with documented provider variance.

### 6.2 Reliability and availability

| Failure injection | Expect |
|---|---|
| LLM timeout / 5xx | Bounded retry; metrics increment |
| Persistent invalid LLM | Terminal reviewable state + error record |
| Redis down at enqueue | Ticket committed; job pending / sweeper re-enqueue |
| Worker crash mid-job | Safe retry; no duplicate finals |
| Prolonged AI outage | Ingestion + agent queue usable (AC-003, NFR-AVL-002) |

### 6.3 Observability smoke

- Structured logs include `request_id` / `ticket_id` / `triage_job_id`
- No JWTs, API keys, or unnecessary PII dumps in logs (NFR-SEC-006)
- Operator can reconstruct path via IDs + `audit_events` + `triage_jobs` (NFR-OBS-003)

---

## 7. Security, Privacy, and Safety Checklist

Run before any external demo.

| Check | Pass criteria | Ref |
|---|---|---|
| Unauthenticated protected routes | `401` | AC-012 |
| Agent on admin mutation | `403` | AC-012 |
| Role matrix | mobile / agent / admin capabilities match [api_contracts.md](./api_contracts.md) §1.4 | NFR-SEC-002/003 |
| Secrets in repo | CI scan clean | AC-013 |
| Secrets in clients | No provider keys in mobile/browser bundles | NFR-SEC-004 |
| TLS in deployed env | HTTPS for clients and provider calls | NFR-SEC-001 |
| XSS / injection | Escaped templates; parameterized SQL; untrusted ticket text in prompts | NFR-SEC-007 |
| MinIO privacy | No public ACLs | system design §7.5 |
| Diagnostics minimization | Only allowlisted fields persisted/forwarded | NFR-PRV-001/004 |
| Retention config | `DATA_RETENTION_DAYS` / `retain_until` present | NFR-PRV-003 |
| Consequential actions | No refund/account mutation from AI output | requirements §5.2 #6 |

---

## 8. UI / UX Quality Gates

Manual or lightweight automated checks against [ui_ux_guidelines.md](./ui_ux_guidelines.md).

### 8.1 Provenance gate (blocking)

- [ ] Customer content, diagnostics, AI inference, and final decision are visually distinct
- [ ] AI diagnosis / likely cause labeled as inferred, never as fact (AC-011)
- [ ] Rule overrides labeled as system/rule source
- [ ] Corrections do not erase AI panel content

### 8.2 Workflow gate

- [ ] Queue → detail → approve/correct → status path completable without dead ends
- [ ] Only legal status transitions offered
- [ ] Low-confidence / review-needed affordance visible when applicable
- [ ] No one-click “AI resolve and send”
- [ ] Loading / empty / error states explicit and recoverable

### 8.3 Accessibility smoke

- [ ] Keyboard path for filters, row open, accept/correct, status change
- [ ] Status/priority not color-only
- [ ] Contrast adequate for chips on chosen theme
- [ ] Form errors associated with fields

### 8.4 Analytics honesty gate

- [ ] Metrics copy states reviewed-only denominators
- [ ] Sample size shown when n is small
- [ ] Unreviewed tickets not implied as confirmed-correct

---

## 9. Data Integrity and Provenance Checks

Align with [data_model.md](./data_model.md).

| Invariant | Verification |
|---|---|
| Separate recoverable facts | Customer, diagnostics, AI, human columns/tables independently readable after correction |
| AI immutability | Update decision does not mutate `ai_triage_results` success row |
| One active triage job | Partial unique / app guard for pending/running |
| Idempotency uniqueness | `(principal_id, key)` unique |
| Controlled values | Final/AI codes ⊆ active taxonomy/teams/priorities |
| Status history | `status_transitions` or equivalent for every change |
| Attachment binaries | Not in Postgres; metadata + MinIO only |

---

## 10. End-to-End Definition of Done Script

Executable scenario (requirements §5.3). Prefer automation; manual acceptable once with evidence.

1. Authenticate as mobile; `POST /tickets` with message + diagnostics → ID returned immediately.
2. Assert ticket persisted; create path did not wait on LLM.
3. Worker processes with stub or live LLM; schema-valid result stored.
4. Agent queue shows category, priority, routing, confidence, status.
5. Agent opens detail; inspects customer, diagnostics, AI separately.
6. Agent corrects at least one field (or approves on a second ticket).
7. Assert AI prediction preserved + correction/approval recorded.
8. Transition ticket to `RESOLVED` then `CLOSED`; audit present.
9. Analytics report agreement/override on reviewed set only.
10. Repeat create with AI disabled; ticket still created and reviewable.

All AC-001–015 must pass for POC acceptance ([system_design.md](./system_design.md) §13).

---

## 11. Defect Severity and Triage

| Severity | Definition | Examples | Release impact |
|---|---|---|---|
| S0 Blocker | Data loss, false durable ack, security hole, AI treated as executed action | Lost ticket after 201; agent can hit admin; secrets committed | Stop release |
| S1 Critical | AC failure or invariant break | Duplicate tickets; AI invents category persisted; override wipes AI | Stop release |
| S2 Major | Feature impaired with workaround | Filter bug; review badge missing; metric mixes unreviewed | Fix before DoD unless waived |
| S3 Minor | Polish / non-blocking UX | Chip contrast; copy nits | Track; do not block POC if ACs green |
| S4 Trivial | Cosmetic | Spacing | Backlog |

**AI-specific defect classes** (tag in issues):

- `ai-schema` — validation/governance failure
- `ai-provenance` — fact/inference confusion
- `ai-eval` — wrong denominator / metric lie
- `ai-degrade` — outage impacts ingestion

---

## 12. CI and Release Gates

### 12.1 Pull request (merge) gate

Must pass:

1. Unit + integration tests for domain, API, worker stubs
2. AC-001–013 and AC-015 automated (or explicitly marked skipped only with issue link — prefer zero skips)
3. Secret scan (AC-013)
4. Migrations apply cleanly on empty DB

### 12.2 POC release / demo gate

Must pass:

1. Merge gate green on release commit
2. Compose E2E DoD script (§10)
3. AC-014 performance smoke on create path
4. Security/privacy checklist (§7)
5. UI provenance + workflow gates (§8.1–8.2)
6. Failure-mode drill: AI down → create still works (AC-003)

### 12.3 Recommended CI job layout

```text
lint/typecheck (if configured)
unit
integration-api (postgres service)
worker-triage (stub LLM)
secrets-scan
e2e-compose (optional on main / nightly)
perf-smoke (nightly or pre-demo)
```

---

## 13. Test Ownership and Evidence

| Area | Owner (role) | Evidence artifact |
|---|---|---|
| Domain/unit | Engineering | pytest report |
| API/AC automation | Engineering | CI logs + case IDs |
| UI provenance | Eng + evaluator | Checklist §8 signed or screenshot set |
| Perf | Engineering | Locust/k6 summary (p95, error rate, lost acks = 0) |
| Live AI spot-check | Evaluator | Small scored sample + notes on override themes |
| Security hygiene | Engineering | gitleaks/CI + checklist §7 |

Store run notes under repo `artifacts/` or CI artifacts only when the team wants durable demo evidence; do not commit secrets or raw customer PII.

---

## 14. Coverage Expectations (POC-practical)

Normative minimum (NFR-MNT-004):

- State machine transitions (legal + illegal)
- API validation and idempotency
- Deterministic rules engine
- AI-output schema + controlled-value validation
- Confidence gating → `NEEDS_REVIEW`
- Override preservation (AI + human)
- Authn/authz matrix smoke
- Analytics reviewed-only denominator

Informative stretch:

- Playwright full console tour
- Property-based tests for tag normalization
- Mutation testing on rules engine
- Large golden-set live model scoring

---

## 15. Open QA Risks

| Risk | Mitigation in tests |
|---|---|
| Provider non-determinism | Stub-first CI; live eval separate |
| Flaky async timing | Explicit job wait helpers; avoid fixed short sleeps where possible |
| UI-only provenance bugs | Assert HTML/ARIA labels + API provenance fields |
| Metric misuse | Dedicated AC-015 tests with mixed reviewed/unreviewed seeds |
| False confidence in soft AI latency | Document NFR-PERF-002 as target; gate on ingestion durability instead |

---

## 16. Document Maintenance

Update this QA spec when:

- acceptance criteria (AC-*) are added/changed;
- API contracts or status machine change;
- provenance model or evaluation denominators change;
- new consequential action surface appears (must remain test-blocked for POC).

**Ownership:** QA criteria here are normative for POC acceptance. Implementation may choose equivalent tooling, but AC-001–015 and the DoD script remain the release bar.
