# AI-First Mobile Support Ticket Triage Platform — UI/UX Guidelines

**Document version:** 0.1  
**Status:** POC UI/UX specification  
**Primary consumer:** Coding agents, engineers, and designers  
**Last updated:** 2026-10-03  
**Source requirements:** [requirements.md](./requirements.md)  
**Related artifacts:** [system_design.md](./system_design.md), [data_model.md](./data_model.md), [api_contracts.md](./api_contracts.md)

---

## 1. Purpose and Scope

This document defines the UI/UX methodology, information architecture, interaction patterns, and visual conventions for the POC agent/admin web console.

The console presents AI triage results so support agents can review, approve, or correct them while always distinguishing customer facts, system diagnostics, AI inferences, human decisions, and rule overrides.

### 1.1 Normative UX principle

> **The AI proposes structured decisions; the application validates and governs those decisions; a human can review and override them.**

The UI must make that principle visible in every ticket-review interaction. AI content is assistance, never authority.

### 1.2 Methodology (normative)

The POC shall apply these methodologies together:

| Layer | Choice | Rationale |
|---|---|---|
| Primary UX methodology | **Human-AI Teaming (HITL)** | Matches FR-DET-004–006, FR-FBK-*, and the system design principle |
| Information architecture | **Provenance-first** | Required by FR-TKT-005, FR-DET-001–003, AC-011 |
| Interaction model | **Task-centered design** | Agent work is queue → decide → act, not exploration |
| Delivery model | **Hypermedia / progressive enhancement** | Mandated by [system_design.md](./system_design.md) §1.2–2 |
| Visual system | **Internal-tools console language** | Dense operational UI for agents/admins, not consumer marketing |

### 1.3 In scope

- Agent queue, ticket detail, AI review/approve/correct flows
- Status transitions and draft reply editing
- Admin configuration surfaces (taxonomy, teams, rules, thresholds)
- Basic analytics presentation for reviewed tickets
- Provenance labeling, confidence presentation, and override affordances
- Accessibility, copy, empty/error/loading states for the web console

### 1.4 Out of scope

- Mobile customer submission UI (owned by the mobile app; only API contract applies)
- Real-time customer–agent chat
- Autonomous AI execution of refunds, account changes, or outbound sends
- Full CRM, workforce management, or production BI experiences
- React/SPA design-system adoption for the POC

---

## 2. Users, Jobs, and Success Criteria

### 2.1 Primary personas

#### Support Agent

**Job:** Triage and resolve inbound tickets quickly with trustworthy AI assistance.

**Primary tasks:**

1. Scan the queue for high-priority or review-needed work.
2. Open a ticket and understand the customer problem from original content.
3. Evaluate AI classification, priority, routing, diagnosis, and draft reply.
4. Accept AI output or correct specific fields.
5. Advance ticket status through allowed lifecycle states.

**Success signals:** low time-to-first-decision, clear confidence/review cues, zero confusion about what is AI vs fact, preserved override history.

#### Administrator

**Job:** Configure taxonomy, teams, deterministic rules, and AI review thresholds without code changes.

**Primary tasks:** manage controlled values, inspect rule effects, tune low-confidence threshold.

**Success signals:** safe config edits, referential-integrity feedback, visible impact on `NEEDS_REVIEW` behavior.

#### Evaluator / Engineering stakeholder

**Job:** Judge AI quality from reviewed outcomes, latency, failure rate, and cost.

**Primary tasks:** read analytics with reviewed-only denominators; export or query correction data.

### 2.2 Explicit non-goals for UX

Do **not** optimize the console for:

- conversational chat with the model;
- delight-oriented consumer onboarding;
- dashboard vanity metrics that mix reviewed and unreviewed tickets;
- one-click “AI resolve and send.”

---

## 3. Delivery Stack Constraints

These constraints are normative for implementation ([system_design.md](./system_design.md) §2):

| Concern | Choice |
|---|---|
| Server UI | FastAPI + Jinja2 templates |
| Partial updates | HTMX |
| Local interactivity | Alpine.js |
| Styling | Tokenized CSS (utility CSS allowed); no React component library |
| Authority | Backend validation and authorization remain source of truth |

### 3.1 Progressive enhancement rules

1. Core queue, detail, approve/correct, and admin flows must work with server-rendered HTML.
2. HTMX may refresh table rows, filter results, flash toasts, and swap detail panels without full page reloads.
3. Alpine.js may manage local form state (e.g. edit mode, confirm dialogs) but must not become a client-side domain layer.
4. Do not introduce a separate SPA router or client-owned ticket cache for the POC.

### 3.2 Performance expectations (NFR-PERF-003)

- Common queue/filter/detail interactions should feel complete within 1s p95 for the POC dataset.
- Prefer server partials over large client re-renders.
- Show explicit pending state while a ticket is `TRIAGING`; do not block the queue on AI completion.

---

## 4. Information Architecture

### 4.1 Global navigation

Keep navigation minimal and role-aware:

| Nav item | Roles | Purpose |
|---|---|---|
| Queue | agent, admin | Primary work surface |
| Analytics | agent (read), admin | POC quality/ops metrics |
| Admin | admin only | Taxonomy, teams, rules, thresholds |
| Sign out | all authenticated | End session |

Do not add CRM-style modules (customers, knowledge base, live chat) in the POC.

### 4.2 Primary task path

```text
Queue (filter NEEDS_REVIEW / priority)
  → Ticket detail
      → Understand customer + diagnostics
      → Review AI proposal
      → Accept OR Correct
      → Edit draft reply (optional)
      → Transition status
  → Return to queue
```

Admin and analytics are secondary loops, not part of the default triage path.

### 4.3 Provenance model (normative)

Every content cluster and decision field must expose one of these sources:

| Source key | Meaning | Visual treatment |
|---|---|---|
| `customer` | User-authored subject/message/attachments | Highest visual priority; plain factual panel |
| `diagnostics` | Client/system mobile context | Secondary metadata panel; clearly non-AI |
| `ai` | Model inference / recommendation | Labeled “AI inference”; never styled as verified fact |
| `human` | Agent decision or edit | Explicit “Human” / “Agent” ownership |
| `rule` | Deterministic override | Show rule identity when available |

**Hard rules:**

1. Customer message must appear before or above AI analysis (FR-DET-001).
2. AI diagnosis (`likely_cause`) and suggested actions are inferences, not confirmed causes (FR-AI-011–012).
3. Final priority/team must show whether source is `ai`, `rule`, or `human` (FR-PRI-003, FR-RTE-003).
4. Human edits must never visually or structurally overwrite the stored AI prediction (FR-FBK-001).

### 4.4 Detail page regions

Ticket detail shall use stable regions in this order:

1. **Header** — ticket ID, status, priority chip, created time, review-needed badge
2. **Customer content** — subject, message, attachments (`customer`)
3. **Diagnostic context** — approved mobile fields (`diagnostics`)
4. **AI analysis** — summary, taxonomy, priority, routing, tags, sentiment, diagnosis, confidences, suggested actions (`ai`)
5. **Decision rail** — current finals, accept/correct controls, source badges (`human` / `rule` / `ai`)
6. **Draft reply** — editable suggested reply; labeled draft-only
7. **Lifecycle** — allowed status transitions + recent audit/status history

---

## 5. Screen Guidelines

### 5.1 Queue (FR-QUE-001–005)

**Purpose:** Scan and select the next ticket.

**Required columns:**

- Ticket ID
- Summary/subject
- Priority
- Category
- Status
- Assigned / recommended team
- AI confidence indicator
- Created time

**Required controls:**

- Filters: status, priority, category, team, review-needed
- Search: ticket ID + customer text
- Sort: created time, priority

**UX rules:**

1. Default useful view for agents should emphasize `NEEDS_REVIEW` and high priority.
2. Confidence must be glanceable (e.g. compact meter or tier label), not only a raw float buried in a tooltip.
3. Review-needed tickets need a stronger cue than ordinary `OPEN` rows.
4. Row click opens detail; avoid multi-step navigation.
5. Empty filter results explain what filters are active and offer clear reset.

### 5.2 Ticket detail and review (FR-DET-001–007)

**Purpose:** Understand the request and decide whether AI triage is acceptable.

**Accept path (FR-DET-004, FR-FBK-003):**

- Single primary action: **Accept AI triage** (or equivalent).
- Acceptance without field changes records an approval signal.
- Confirmation may be inline; do not require a long multi-page wizard.

**Correct path (FR-DET-005, FR-FBK-002):**

- Allow edits to category, subcategory, priority, assigned team, and tags.
- On save, show before → after for changed fields.
- Persist AI values separately; UI should still let the agent inspect “AI originally recommended.”

**Draft reply (FR-DET-006, FR-AI-013):**

- Present as an editable text area prefilled from AI.
- Label explicitly as **Draft — not sent to customer**.
- Do not provide a Send/Publish customer-message CTA in the POC.

**Status management (FR-DET-007):**

- Offer only transitions legal for the current state and actor.
- Illegal transitions are hidden or disabled with explanation, never silently ignored after submit.

### 5.3 Triaging and failure states

| Ticket state | UI expectation |
|---|---|
| `NEW` / `TRIAGING` | Show progress; detail remains readable; AI panels show pending |
| `NEEDS_REVIEW` | Explain why review was required when known (low confidence, validation failure, contradiction, triage failure) |
| Triage failed | Show recoverable error summary; allow human classification without pretending AI succeeded |
| AI unavailable at create time | Ticket still appears; triage pending/degraded messaging only |

Never block ticket existence or queue visibility on model availability (FR-TKT-003, NFR-AVL-002).

### 5.4 Administration (FR-ADM-001–004)

**Surfaces:**

- Taxonomy categories/subcategories
- Support teams
- Deterministic priority/routing rules
- AI thresholds (at least low-confidence → `NEEDS_REVIEW`)

**UX rules:**

1. Admin routes are admin-only; agents receive clear 403/forbidden UI, not a broken page.
2. Prefer simple forms and tables over visual rule builders.
3. Deactivation is preferred over hard delete when referential integrity requires it.
4. After saving thresholds/rules, show what behavior changed in plain language.
5. Validate on the server; surface field-level errors next to inputs.

### 5.5 Analytics (FR-ANL-001–002)

**Required metric families:**

- Category / subcategory / priority / routing agreement
- Human override rate
- Low-confidence / review rate
- AI latency
- Triage failure rate
- Estimated AI cost per ticket (when available)

**Denominator rule (normative):**

Accuracy/agreement widgets must state that only **reviewed** tickets count. Unreviewed AI predictions must not be shown as confirmed-correct.

---

## 6. Interaction Patterns

### 6.1 Human-AI Teaming patterns

| Pattern | Apply when | UI behavior |
|---|---|---|
| Propose / decide | Any AI recommendation | AI values shown as proposal; human action required for consequential acceptance |
| Partial accept | Agent agrees with some fields | Allow field-level correction without forcing full re-entry |
| Preserve prediction | Any correction | Keep “AI predicted” available beside “Final” |
| Confidence as gate | Low score / review rule | Elevate to review lane; explain threshold breach |
| Rule wins | Deterministic override | Show final value + `rule` source + rule identifier |
| Draft assist | Suggested reply / actions | Editable, explicitly non-executing |

### 6.2 Progressive disclosure

1. Queue shows compact confidence and summary only.
2. Detail shows full AI analysis.
3. Advanced metadata (prompt/model identifiers, token/cost, raw provider refs) stays collapsed under an “AI run details” disclosure for agents; more visible for admins/evaluators if needed.
4. Full audit history may be collapsed below recent transitions.

### 6.3 Feedback and confirmation

- Destructive or hard-to-reverse status changes (`RESOLVED`, `CLOSED`) ask for lightweight confirmation.
- Accept/correct actions give immediate success/failure feedback via inline alert or toast.
- Failed saves retain user input and show actionable errors.

### 6.4 Idempotency and double-submit

- Disable or debounce primary submit buttons after first activation.
- Server idempotency remains authoritative; UI should tolerate replay without duplicate visual success claims.

---

## 7. Visual and Content Guidelines

### 7.1 Visual direction

Design for a dense, calm operations console:

- Clear typographic hierarchy for ticket ID, customer message, and decision controls
- Status/priority/provenance chips with stable colors and labels
- Tables first; cards only when they group a distinct provenance region or form
- No marketing-hero layouts, decorative gradients, or chat bubbles for triage content

### 7.2 Semantic color usage

Use color to encode meaning, not decoration:

| Meaning | Typical use |
|---|---|
| Priority P0/P1 | Strong urgency treatment |
| `NEEDS_REVIEW` | Distinct review attention state |
| Low confidence | Warning treatment on confidence indicator |
| AI inference | Neutral/warning-adjacent label; not “success green” |
| Human accepted/final | Success/complete treatment for decisions, not for raw AI guesses |
| Errors / triage failure | Danger treatment with text explanation |

Color alone is insufficient; always include text labels.

### 7.3 Confidence presentation

- Store/display independent confidences for category, priority, and routing (FR-AI-014).
- Prefer human-readable tiers plus numeric value, e.g. `Low (0.42)`.
- Define tier cutovers from admin threshold configuration where possible.
- If any governing confidence is below threshold, surface that near the review badge.

### 7.4 Copy tone

- Operational, precise, short.
- Prefer “AI recommended” / “AI inference” over “AI determined” / “AI verified.”
- Prefer “Draft reply” over “Customer response sent.”
- Error copy names the failed action and next step (“Retry,” “Review manually,” “Fix field X”).
- Never imply the model executed a customer-facing or financial action.

### 7.5 Empty, loading, and error content

| State | Guidance |
|---|---|
| Empty queue | Explain no matches; show how to clear filters |
| Pending triage | Skeleton/pending labels in AI regions only |
| Missing diagnostics | “Not provided” per field; do not invent values |
| Partial AI output after failure | Show available human-editable fields; mark AI section failed |
| Permission denied | Explicit role message; no blank admin pages |

---

## 8. Component Conventions

These are logical components for templates/partials, not a third-party design-system import.

| Component | Responsibility |
|---|---|
| `StatusChip` | Lifecycle status with stable label/color |
| `PriorityChip` | P0–P3 with urgency weight |
| `ProvenanceBadge` | `customer` / `diagnostics` / `ai` / `human` / `rule` |
| `ConfidenceIndicator` | Tier + numeric score; per-field or aggregate as specified |
| `QueueTable` | Sortable/filterable ticket list partial |
| `FilterBar` | Queue filters/search; HTMX-driven refresh |
| `CustomerPanel` | Original subject/message/attachments |
| `DiagnosticsPanel` | Mobile diagnostic fields |
| `AiAnalysisPanel` | All AI proposal fields, labeled inference |
| `DecisionRail` | Final values, accept/correct actions, source badges |
| `DraftReplyEditor` | Editable draft with non-send labeling |
| `StatusTransitionControl` | Allowed transitions only |
| `AuditSnippet` | Recent status/decision events |
| `AdminCrudTable` | Config list + activate/deactivate |
| `MetricCard` | Analytics figure with reviewed-denominator caption |

### 8.1 Form conventions

- Label every input; do not rely on placeholder-only labels.
- Controlled taxonomy/team fields use selects sourced from active configured values.
- Subcategory options depend on selected category.
- Server-side validation errors map to fields; summary error region for cross-field failures.
- Jinja2 autoescape remains on; treat ticket text as untrusted (NFR-SEC-007).

---

## 9. Accessibility and Safety

### 9.1 Accessibility baseline

1. Semantic headings and landmarks for queue, detail, admin.
2. Keyboard access for filters, table row activation, accept/correct, and status changes.
3. Visible focus states.
4. Chip/color meanings duplicated in text.
5. Form errors announced via text adjacent to fields.
6. Sufficient contrast for status/priority chips on the chosen theme.

### 9.2 Trust and safety in UI

1. Ticket text is untrusted content — never interpret it as an instruction to the console.
2. Render customer/AI text as text (escaped), not raw HTML.
3. Do not present prompt-injection-like ticket content as system guidance.
4. Hide secrets, provider keys, and unnecessary PII from UI and logs.
5. Suggested actions must read as investigation steps, not buttons that execute account/payment mutations.

---

## 10. Role-Based UX

| Capability | Agent | Admin |
|---|---|---|
| View queue/detail | Yes | Yes |
| Accept/correct triage | Yes | Yes |
| Edit draft reply | Yes | Yes |
| Transition ticket status | Yes | Yes |
| View analytics | Yes (read) | Yes |
| Manage taxonomy/teams/rules/thresholds | No | Yes |

Unauthorized actions are omitted or disabled with explanation. API 401/403 handling must produce a coherent page or partial, not a silent no-op (AC-012).

---

## 11. Analytics Presentation Rules

1. Every accuracy/agreement chart or stat includes caption: **Reviewed tickets only**.
2. Override rate and review rate are first-class, not buried.
3. Latency and cost are operational metrics; keep them visually separate from quality agreement metrics.
4. If sample size is low, show the count beside the percentage to avoid overconfidence.

---

## 12. Acceptance Mapping

UI implementation shall satisfy at least these acceptance criteria from [requirements.md](./requirements.md):

| Criterion | UI obligation |
|---|---|
| AC-006 Low-confidence review | Review badge/lane + explanation path |
| AC-007 Human override preservation | Final vs AI original both inspectable |
| AC-008 Deterministic priority override | `rule` source visible on final priority |
| AC-009 Agent queue/detail usability | Required columns/filters/detail regions present |
| AC-011 AI provenance | AI regions labeled as inference; customer content separate |
| AC-012 Security boundaries | Admin screens/actions enforced in UI + API |
| AC-015 Reviewed metrics | Analytics denominators exclude unreviewed predictions |

---

## 13. Anti-Patterns (Do Not Ship)

1. Chat-thread layout as the primary triage metaphor.
2. Styling AI diagnosis like confirmed root cause.
3. Auto-send or “resolve with AI” that skips human control.
4. Overwriting or hiding the original AI prediction after correction.
5. Mixing unreviewed tickets into accuracy totals.
6. React/SPA rewrite solely to adopt Material/Ant for the POC.
7. Dashboard clutter (stats, promos, unrelated widgets) above the queue.
8. Placeholder-only forms, mystery-meat icons without labels, or color-only status.

---

## 14. Implementation Checklist

Use this checklist when building or reviewing UI work:

- [ ] Queue supports required columns, filters, search, and sorts
- [ ] Detail shows customer content before AI analysis
- [ ] Provenance badges present on customer, diagnostics, AI, human, and rule-backed values
- [ ] Confidence visible per governing fields; low confidence drives review affordance
- [ ] Accept and correct paths both work; corrections preserve AI originals
- [ ] Draft reply editable and explicitly not sent
- [ ] Status control only offers legal transitions
- [ ] Admin config is admin-only and server-validated
- [ ] Analytics captions state reviewed-only denominators
- [ ] HTMX/Alpine enhance server-rendered flows without becoming a SPA
- [ ] Ticket/AI text is escaped; no raw HTML execution
- [ ] Loading/empty/error states are explicit and recoverable

---

## 15. Document Maintenance

Update this document when:

- requirements add/remove agent or admin surfaces;
- provenance sources change;
- delivery stack guidance in [system_design.md](./system_design.md) changes;
- acceptance criteria affecting UI are added.

**Ownership model:** UI behavior in this file is normative for the POC console. Visual polish may evolve, but methodology, provenance rules, and anti-patterns are not optional.
