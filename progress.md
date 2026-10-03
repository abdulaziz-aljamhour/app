# Progress Log

## 2026-10-03 16:23 +03 (2026-10-03T13:23:15Z) — Artifact summary

All seven `artifacts/` docs are **v0.1**, last updated **2026-10-03**, and describe one POC: an **AI-first mobile support ticket triage platform**. Shared normative principle across design, UX, and QA:

> The AI proposes structured decisions; the application validates and governs those decisions; a human can review and override them.

Consequential actions (refunds, account changes, outbound customer replies) are never executed by AI.

---

### Cross-cutting picture

| Aspect | Summary |
|---|---|
| **Problem** | Mobile users submit support requests; AI structures them; agents review/override in a web console |
| **Roles** | `mobile` (ingest), `agent` (queue/review), `admin` (taxonomy/teams/rules/config) |
| **Stack** | Python-first: FastAPI + Jinja2/HTMX/Alpine, SQLAlchemy/Alembic/PostgreSQL, Celery/Redis, MinIO, JWT, OpenAI-compatible structured JSON |
| **Delivery** | 6 one-week sprints, **48 stories / 139 points**, covering AC-001–015 |
| **Provenance** | Always separate `customer` / `diagnostics` / `ai` / `rule` / `human` facts |

---

### Per-artifact summary

#### 1. `requirements.md` — POC build specification
- Business goals: cut manual triage, classify issues, recommend priority/team, use diagnostics, keep provenance clear, allow overrides, capture corrections, measure quality/latency/cost.
- **In scope:** ingest API, diagnostics + attachments metadata, async AI triage, taxonomy/priority, agent queue + HITL review, admin config, basic analytics/audit.
- **Out of scope:** multi-tenant SaaS, full CRM, omnichannel, autonomous consequential actions, enterprise RBAC, custom model training, large RAG, CRM integrations, production BI, live chat.
- Functional clusters: FR-TKT, FR-STA, FR-TAX, FR-AI, FR-PRI, FR-RTE, FR-QUE, FR-DET, FR-FBK, FR-ADM, FR-ANL, FR-AUD (+ NFRs for perf/reliability/security/privacy/observability/maintainability).
- **15 acceptance criteria** (AC-001–015) and a recommended build order define Done.

#### 2. `system_design.md` — Architecture & components
- Containers: FastAPI API + Celery worker + Postgres + Redis + MinIO; LLM via provider-agnostic port.
- Flow: durable ticket create → enqueue triage → worker validates structured AI → rules may override → agent queue/`NEEDS_REVIEW` when low confidence.
- Packages: domain / application / infrastructure / schemas; services for ingestion, lifecycle, worker, AI port, rules, queue/detail, feedback, admin, analytics.
- Covers authz, secrets, logging hygiene, failure modes (AI/Redis down must not block ingest), observability, Compose topology, testing, and design-aligned Definition of Done.

#### 3. `data_model.md` — Logical schema
- Postgres is system of record; MinIO holds attachment binaries only.
- Core entities: users, tickets, diagnostics, attachments, taxonomy, teams, triage_jobs, ai_triage_results (immutable after success), ticket_decisions, feedback_corrections, approvals, status_transitions, rules, app_config, audit_events, idempotency_keys.
- Status machine: `NEW → TRIAGING → NEEDS_REVIEW | OPEN → IN_PROGRESS → RESOLVED → CLOSED`; priorities P0–P3.
- Integrity rules keep AI predictions separate from human finals; seed teams/taxonomy/users for POC.

#### 4. `api_contracts.md` — Interface contract
- Base: `/api/v1`, JWT roles, idempotency on create, standard error envelope.
- Surfaces: mobile create/get (+ optional upload); agent queue/detail/approve/correct/status; admin taxonomy/teams/rules/config; analytics/export; auth login helper.
- Defines AI structured-output schema + post-schema validation; Jinja/HTMX web routes are non-JSON companions.
- Create must not 503 solely because the LLM is down.

#### 5. `ui_ux_guidelines.md` — Agent/admin console UX
- HITL + provenance-first + task-centered (queue → decide → act); hypermedia progressive enhancement (not React SPA).
- Screens: queue, ticket detail/review with provenance panels, triaging/failure states, admin, analytics.
- AI never presented as customer fact; confidence and override affordances required; internal-tools visual language.
- Mobile customer UI out of scope (API only).

#### 6. `quality_assurance.md` — Verification & gates
- Objectives: durable ingest, governed AI, human control, provenance clarity, measurable quality, safe ops, usable under load (e.g. ingest p95 ≤ 1s).
- Pyramid: unit / API / worker / contract / E2E; stub LLM by default; live AI optional for demos.
- Traceability for AC-001–015; suites for ingest, lifecycle/audit, AI, rules, queue/detail, admin, analytics/feedback; UI provenance gate is blocking.
- CI merge + POC release gates; severity model and coverage expectations for a POC-practical bar.

#### 7. `sprints.md` — Countable backlog
| Sprint | Theme | Stories | Points | Unlocks |
|---|---|---:|---:|---|
| S1 | Foundation, schema, auth | 8 | 24 | AC-013 |
| S2 | Ingestion, lifecycle, audit | 8 | 24 | AC-001–003, AC-010 |
| S3 | Async AI + rules + governance | 9 | 26 | AC-004–006, AC-008 |
| S4 | Agent queue + provenance UI | 8 | 23 | AC-009, AC-011–012 |
| S5 | HITL approve/correct + admin | 8 | 22 | AC-007 |
| S6 | Analytics, hardening, DoD | 7 | 20 | AC-014–015 |
| **Total** | | **48** | **139** | **15/15 ACs** |

Serial dependency: S1 → S2 → S3 → S4 → S5 → S6. Progress formulas: story %, point % of 139, AC % of 15.

---

### Current project state (from artifacts)

- Spec set is complete and internally cross-linked; no implementation progress is recorded in these docs.
- Natural next step per sprint plan: **S1 Foundation** (Compose monorepo, migrations, seeds, JWT, domain status machine).
