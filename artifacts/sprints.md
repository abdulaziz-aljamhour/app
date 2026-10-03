# Sprint Plan — AI Ticket Triage POC

**Document version:** 0.1  
**Status:** Countable sprint backlog for POC delivery  
**Last updated:** 2026-10-03  
**Source artifacts:** [requirements.md](./requirements.md), [system_design.md](./system_design.md), [data_model.md](./data_model.md), [api_contracts.md](./api_contracts.md), [ui_ux_guidelines.md](./ui_ux_guidelines.md), [quality_assurance.md](./quality_assurance.md)

---

## 1. How to count this plan

| Rule | Value |
|---|---|
| Sprint length | **1 week** (5 engineering days) |
| Sprint count | **6** (S1–S6) |
| Work unit | **Story** with fixed ID `S{n}-{nn}` |
| Story size | Fibonacci points: 1, 2, 3, 5, 8 |
| Team assumption | **1–2 engineers** full-time on the POC |
| Capacity target | **~20–26 points / sprint** (adjust if team size changes) |
| Done means | Sprint exit criteria + linked ACs green (stub LLM unless noted) |

**Totals (countable):**

| Metric | Count |
|---|---|
| Sprints | 6 |
| Stories | 48 |
| Story points | 139 |
| Acceptance criteria covered | AC-001–015 (all) |
| Functional requirement clusters | 12 (FR-TKT … FR-AUD) |

Progress formula:

```text
sprint_completion_% = done_stories / committed_stories
poc_completion_%    = sum(done_points) / 139
ac_completion_%     = passed_ACs / 15
```

---

## 2. Sprint map (at a glance)

| Sprint | Theme | Stories | Points | Primary ACs unlocked |
|---|---|---|---:|---|
| **S1** | Foundation, domain, schema, auth | 8 | 24 | AC-013 |
| **S2** | Ticket ingestion + lifecycle + audit | 8 | 24 | AC-001, AC-002, AC-003, AC-010 |
| **S3** | Async AI triage + rules + governance | 9 | 26 | AC-004, AC-005, AC-006, AC-008 |
| **S4** | Agent queue + provenance detail UI | 8 | 23 | AC-009, AC-011, AC-012 |
| **S5** | HITL approve/correct + admin config | 8 | 22 | AC-007 |
| **S6** | Analytics, hardening, DoD release | 7 | 20 | AC-014, AC-015 (+ full suite) |
| | **Total** | **48** | **139** | **15/15** |

Dependency chain (must stay serial at sprint boundaries):

```text
S1 → S2 → S3 → S4 → S5 → S6
         ↘ rules seed used by S3
                    ↘ feedback used by S5/S6 metrics
```

Aligned with requirements §5.5 / system design §12 build order, compressed into countable weekly batches.

---

## 3. Sprint 1 — Foundation

**Goal:** Runnable Compose skeleton, versioned schema, seed data, auth, and domain primitives. No triage yet.

**Points:** 24 · **Stories:** 8 · **Unblocks:** S2 ingestion

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S1-01 | Scaffold monorepo layout (`apps/api`, `apps/worker`, `packages/{domain,application,infrastructure,schemas}`) + Docker Compose (`api`, `worker`, `postgres`, `redis`, `minio`) | 5 | system_design §2, §4, §10 |
| S1-02 | SQLAlchemy models + Alembic migrations for core tables: `users`, `tickets`, `ticket_diagnostics`, `attachments`, `taxonomy_*`, `support_teams`, `idempotency_keys`, `audit_events`, `status_transitions`, `app_config`, `triage_jobs` stubs | 8 | data_model §5; NFR-MNT-003 |
| S1-03 | Seed FR taxonomy + teams (`account_support`, `billing_support`, `technical_support`, `product_support`) + POC users (`mobile`, `agent`, `admin`) | 2 | FR-TAX-002, FR-RTE-002 |
| S1-04 | Domain status machine (legal transitions only) + provenance enums | 3 | FR-STA-001–006; data_model §3–4 |
| S1-05 | JWT auth helper (`POST /api/v1/auth/login`) and role claims `mobile` \| `agent` \| `admin` | 3 | api_contracts §6; NFR-SEC-002/003 |
| S1-06 | structlog + `X-Request-Id` middleware; env-based secrets loading | 1 | NFR-OBS-001, NFR-SEC-004 |
| S1-07 | pytest harness + factories skeleton; gitleaks/CI secrets scan | 1 | NFR-MNT-004; AC-013 |
| S1-08 | README: local `compose up`, migrate, seed, obtain tokens | 1 | — |

### Exit criteria

- [ ] `docker compose up` boots api/worker/postgres/redis/minio
- [ ] Migrations apply on empty DB; seed loads taxonomy/teams/users
- [ ] Login returns JWT for each role
- [ ] Illegal status transition rejected in unit tests
- [ ] Secret scan clean (**AC-013**)

### Demo

Compose stack healthy; show seeded taxonomy rows and three role tokens.

---

## 4. Sprint 2 — Ingestion & lifecycle

**Goal:** Durable ticket create before AI; idempotency; status audit; AI-outage does not block create.

**Points:** 24 · **Stories:** 8 · **Unblocks:** S3 worker

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S2-01 | `POST /api/v1/tickets` — validate message/subject/diagnostics allowlist; persist `NEW` in one transaction | 5 | FR-TKT-001–006; AC-001 |
| S2-02 | Idempotency-Key scoped to principal; replay + body-hash conflict `409` | 5 | FR-TKT-007; AC-002 |
| S2-03 | Enqueue `triage_jobs` after commit; if Redis down, keep ticket + `pending` job (sweeper path stub OK) | 3 | FR-TKT-003; NFR-REL-001; AC-003 |
| S2-04 | `GET /api/v1/tickets/{id}` customer-visible subset | 2 | api_contracts §2.2 |
| S2-05 | Agent/system status transition service + `status_transitions` / `audit_events` writer | 3 | FR-STA-*; FR-AUD-*; AC-010 |
| S2-06 | Attachment metadata accept + optional MinIO presign (images only) | 3 | FR-TKT-001 attachments; api_contracts §2.3 |
| S2-07 | Authz: mobile can create; unauthenticated → `401` | 1 | AC-012 (partial) |
| S2-08 | Automated `QA-TKT-01..09`, `QA-STA-01`, `QA-AUD-01` | 2 | quality_assurance §5.1–5.2 |

### Exit criteria

- [ ] **AC-001**, **AC-002**, **AC-003**, **AC-010** pass
- [ ] Create p95 path does not call LLM
- [ ] Ticket visible when worker/LLM stopped

### Demo

Create ticket with diagnostics → immediate ID; retry same key → same ID; kill Redis/worker → create still 201.

---

## 5. Sprint 3 — AI triage pipeline

**Goal:** Async structured triage with schema governance, retries, confidence gates, and deterministic rule overrides.

**Points:** 26 · **Stories:** 9 · **Unblocks:** S4 agent UI

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S3-01 | Celery worker task `triage.process_ticket`; claim job; `NEW → TRIAGING` | 3 | FR-AI-001; FR-STA-003 |
| S3-02 | `TriageLLMPort` + stub adapter (CI) + OpenAI-compatible adapter | 5 | FR-AI-002–003; NFR-MNT-002 |
| S3-03 | Pydantic AI output schema + controlled-value validation (taxonomy/team/priority) | 3 | FR-AI-004–014; FR-TAX-001; AC-004/005 |
| S3-04 | Persist immutable `ai_triage_results` + initial `ticket_decisions`; raw/metadata fields | 3 | FR-AI-015–016; FR-FBK-001 |
| S3-05 | Bounded retries, timeouts, terminal `NEEDS_REVIEW` + error record; stuck-job reaper | 3 | NFR-REL-002–004; AC-005 |
| S3-06 | Low-confidence / invalid / failure → `NEEDS_REVIEW`; else `OPEN` | 2 | FR-STA-004–005; AC-006 |
| S3-07 | Deterministic rules engine (priority/routing override; sentiment≠priority) | 3 | FR-PRI-002–004; FR-RTE-003; AC-008 |
| S3-08 | Tag normalization; prompt treats ticket text as untrusted; minimize provider payload | 2 | FR-AI-009; NFR-SEC-007; NFR-PRV-004 |
| S3-09 | Worker tests `QA-AI-01..12`, `QA-PRI-*`, `QA-RTE-01` with stub LLM | 2 | quality_assurance §5.3–5.4 |

### Exit criteria

- [ ] **AC-004**, **AC-005**, **AC-006**, **AC-008** pass under stub LLM
- [ ] Invented taxonomy never persisted as success
- [ ] Rule match sets `priority_source=rule`
- [ ] Worker redelivery does not duplicate active AI result

### Demo

Golden tickets: billing activation → structured fields; bad stub → review; low confidence → `NEEDS_REVIEW`; rule forces P1.

---

## 6. Sprint 4 — Agent console

**Goal:** Task-centered HITL queue and provenance-first ticket detail (Jinja2 + HTMX + Alpine.js).

**Points:** 23 · **Stories:** 8 · **Unblocks:** S5 review actions

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S4-01 | Agent session/login UI + role-aware nav (Queue / Analytics placeholder / Admin placeholder) | 2 | ui_ux §4.1; FR-QUE-001 |
| S4-02 | Queue API `GET /api/v1/agent/tickets` + HTMX table (required columns) | 5 | FR-QUE-002; api_contracts §3.1 |
| S4-03 | Filters (status, priority, category, team, needs_review), search, sort | 3 | FR-QUE-003–005; AC-009 |
| S4-04 | Ticket detail API + page: customer / diagnostics / AI / decision panels | 5 | FR-DET-001–003; AC-011 |
| S4-05 | Confidence chips + review-needed badge; TRIAGING pending state | 2 | ui_ux guidelines; FR-AI-014 |
| S4-06 | Status transition control (legal transitions only) wired to API | 3 | FR-DET-007; FR-STA-* |
| S4-07 | Authz matrix smoke: agent OK on queue; admin-only routes still 403 for later | 1 | AC-012 |
| S4-08 | UI provenance gate automation (HTML/ARIA labels) + `QA-QUE-*`, `QA-DET-01/05` | 2 | quality_assurance §8.1; AC-011 |

### Exit criteria

- [ ] **AC-009**, **AC-011** pass; **AC-012** unauthenticated/agent boundaries for shipped routes
- [ ] Queue → detail path works without SPA
- [ ] AI diagnosis labeled as inference, never as customer fact

### Demo

Filter `NEEDS_REVIEW`, open ticket, show four provenance panels side by side.

---

## 7. Sprint 5 — Human review & admin

**Goal:** Agents approve/correct without erasing AI; admins configure taxonomy, teams, rules, thresholds.

**Points:** 22 · **Stories:** 8 · **Unlocks:** measurable evaluation in S6

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S5-01 | `POST .../approve` + UI accept flow; approval signal when unchanged | 3 | FR-DET-004; FR-FBK-003 |
| S5-02 | `PATCH .../decision` corrections for category/subcategory/priority/team/tags | 5 | FR-DET-005; FR-FBK-001–002; AC-007 |
| S5-03 | Suggested reply edit (draft only; no send) | 2 | FR-DET-006 |
| S5-04 | Admin taxonomy CRUD UI/API with referential-integrity feedback | 3 | FR-ADM-001; FR-TAX-003 |
| S5-05 | Admin teams + rules + `LOW_CONFIDENCE_THRESHOLD` config | 3 | FR-ADM-002–004; NFR-MNT-001 |
| S5-06 | Admin-only authz enforcement (agent → `403`) | 2 | AC-012; NFR-SEC-003 |
| S5-07 | Correction/audit rows for consequential edits | 2 | FR-AUD-001–002; QA-FBK-01 |
| S5-08 | Tests `QA-DET-02..04`, `QA-ADM-01..03`, AC-007 | 2 | quality_assurance §5.5–5.6 |

### Exit criteria

- [ ] **AC-007** and remaining **AC-012** admin mutations pass
- [ ] After correction, AI prediction and human final both queryable
- [ ] Threshold/rule change affects next triage without code change

### Demo

Correct `TECHNICAL` → `BILLING`; show AI panel unchanged + correction metadata; admin lowers confidence threshold and re-run fixture ticket.

---

## 8. Sprint 6 — Analytics, hardening, DoD

**Goal:** Reviewed-only metrics, ops observability, performance smoke, and full POC Definition of Done.

**Points:** 20 · **Stories:** 7 · **Unlocks:** POC release / demo gate

### Stories

| ID | Story | Pts | Refs |
|---|---|---:|---|
| S6-01 | `GET /api/v1/analytics/overview` — agreement, override, review rate, latency, failure, cost | 5 | FR-ANL-001–002; AC-015 |
| S6-02 | Evaluation CSV export + Analytics page with reviewed-only copy / sample size | 3 | FR-FBK-004; ui_ux §8.4 |
| S6-03 | Prometheus `/metrics` for API + triage counters/histograms | 2 | NFR-OBS-002; system_design §9 |
| S6-04 | Locust/k6 create-path smoke; assert p95 ≤ 1s and zero lost acks under POC burst | 3 | AC-014; NFR-PERF-001/004 |
| S6-05 | Compose E2E DoD script (requirements §5.3 / QA §10) automated | 3 | DoD; AC-001–015 |
| S6-06 | Security/privacy checklist + logging hygiene pass | 2 | quality_assurance §7; NFR-SEC/PRV |
| S6-07 | UI workflow + accessibility smoke; close open S2–S5 test gaps | 2 | quality_assurance §8.2–8.3 |

### Exit criteria

- [ ] **AC-014**, **AC-015** pass
- [ ] Full **AC-001–015** green on release commit
- [ ] Compose DoD script succeeds with AI disabled path included
- [ ] Merge + demo gates in quality_assurance §12 satisfied

### Demo

End-to-end: mobile create → triage → agent correct → resolve/close → analytics on reviewed set only; then AI-down create still works.

---

## 9. Requirement → sprint allocation

| Cluster | Sprint(s) | Notes |
|---|---|---|
| FR-TKT-001–007 | S2 | Ingestion durability first |
| FR-STA-001–006 | S1 (machine), S2–S4 (apply) | |
| FR-TAX-001–003 | S1 seed, S3 validate, S5 admin | |
| FR-AI-001–016 | S3 | |
| FR-PRI-001–004 | S3 (+ S5 admin rules) | |
| FR-RTE-001–003 | S1 seed, S3 assign, S5 admin | |
| FR-QUE-001–005 | S4 | |
| FR-DET-001–007 | S4–S5 | Detail in S4; approve/correct in S5 |
| FR-FBK-001–004 | S3 persist, S5 correct, S6 export | |
| FR-ADM-001–004 | S5 | |
| FR-ANL-001–002 | S6 | |
| FR-AUD-001–002 | S2 (+ ongoing) | |
| NFR-PERF / REL / AVL | S2–S3, hard gate S6 | |
| NFR-SEC / PRV / OBS / MNT | S1 continuous; checklist S6 | |

### AC burndown by sprint end

| After sprint | ACs expected green |
|---|---|
| S1 | AC-013 |
| S2 | + AC-001, AC-002, AC-003, AC-010 → **5** |
| S3 | + AC-004, AC-005, AC-006, AC-008 → **9** |
| S4 | + AC-009, AC-011, AC-012 → **12** |
| S5 | + AC-007 → **13** |
| S6 | + AC-014, AC-015 → **15** |

---

## 10. Sprint board columns (suggested)

Use any tracker; keep these columns so progress stays countable:

```text
Backlog → Sprint committed → In progress → Review → Done
```

Each story card should carry: `S{n}-{nn}`, points, FR/AC IDs, and a single checkbox Definition of Done line from the sprint exit list.

Optional labels: `api`, `worker`, `ui`, `qa`, `infra`, `ai-schema`, `ai-provenance`.

---

## 11. Risks & stretch (not in the 139)

| Item | Why deferred | Revisit |
|---|---|---|
| Live-LLM nightly eval | Stub covers merge ACs | After S6 demo |
| Playwright full console tour | HTMX assertions sufficient for POC | Stretch |
| Attachment malware scanning | Explicitly out of POC QA | Post-POC |
| Celery beat packaging polish | Reaper can be API cron initially | S3/S6 as needed |
| Multilingual triage | Assumptions §5.1 | Out of scope |

If a sprint overruns: **cut optional attachment binary path (S2-06)** or **presign polish** before cutting AC-linked stories. Do not cut AC-linked stories without moving the AC on the burndown table.

---

## 12. Tracking template

Copy per sprint retro:

```markdown
## Sprint N retrospective snapshot
- Committed stories: __ / __
- Committed points: __ / __
- ACs newly green: __
- Carry-over IDs: __
- Blockers: __
```

POC burn-up checkpoint (update weekly):

| Checkpoint | Stories done | Points done | ACs green |
|---|---:|---:|---:|
| End S1 | | | |
| End S2 | | | |
| End S3 | | | |
| End S4 | | | |
| End S5 | | | |
| End S6 (target) | 48 | 139 | 15 |

---

## 13. Document maintenance

Update this file when:

- a story is split/merged (keep total points honest);
- an AC moves between sprints;
- capacity assumption (team size) changes — rescale points/sprint, not AC order.

**Ownership:** Sprint scope is informative for scheduling; **AC-001–015 and requirements §5.3 DoD remain the release bar** ([quality_assurance.md](./quality_assurance.md) §12).
