# API Contracts — AI Ticket Triage POC

**Document version:** 0.1  
**Status:** POC interface contract  
**Last updated:** 2026-10-03  
**Parent design:** [system_design.md](./system_design.md)  
**Data model:** [data_model.md](./data_model.md)  
**Source requirements:** [requirements.md](./requirements.md) §4.1–4.3, FR-AI-003

Base URL prefix: `/api/v1`  
Content type: `application/json` unless noted (HTMX UI may use form/`text/html` on non-API routes).

---

## 1. Conventions

### 1.1 Authentication

| Header / cookie | Usage |
|---|---|
| `Authorization: Bearer <jwt>` | Mobile, agent, and admin JSON APIs |
| Session cookie (optional) | Browser HTMX console after login |

JWT claims (minimum): `sub`, `role` (`mobile` \| `agent` \| `admin`), `exp`.

### 1.2 Common headers

| Header | Required | Notes |
|---|---|---|
| `Idempotency-Key` | Yes on `POST /tickets` | Client-generated; scoped to principal |
| `Authorization` | Yes on protected routes | |
| `X-Request-Id` | Optional | Echoed; generated if absent |

### 1.3 Error shape

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed",
    "details": [
      {"field": "message", "issue": "must be a non-empty string"}
    ]
  }
}
```

| HTTP | When |
|---|---|
| 400 | Malformed JSON / bad headers |
| 401 | Missing/invalid auth |
| 403 | Authenticated but insufficient role |
| 404 | Unknown ticket/resource |
| 409 | Conflict (illegal status transition) |
| 422 | Semantic validation failure |
| 429 | Optional rate limit |
| 500 | Unexpected server error |
| 503 | Dependency unavailable for **non-ingestion** paths when applicable |

Ticket **creation** must not return 503 solely because the LLM is down.

### 1.4 Role matrix

| Capability | mobile | agent | admin |
|---|---|---|---|
| Create ticket | yes | no* | no* |
| Queue / detail / review | no | yes | yes |
| Status transitions | no | yes | yes |
| Taxonomy/teams/rules/config | no | no | yes |
| Analytics | no | read | read |

\*POC may allow admin token for test harness create; production-like split keeps create on mobile credentials.

---

## 2. Mobile Ingestion API

### 2.1 Create ticket

```http
POST /api/v1/tickets
Authorization: Bearer <mobile_jwt>
Idempotency-Key: <client-generated-key>
Content-Type: application/json
```

**Request body:**

```json
{
  "subject": "Premium not activated",
  "message": "I paid for premium but the app still says I am on the free plan.",
  "attachments": [
    {
      "filename": "screenshot.png",
      "content_type": "image/png",
      "size_bytes": 184220,
      "storage_key": "optional-if-already-uploaded"
    }
  ],
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
    "recent_error_codes": ["SUBSCRIPTION_SYNC_FAILED"],
    "diagnostics_opt_in": true
  }
}
```

| Field | Rule |
|---|---|
| `message` | required, non-empty string |
| `subject` | optional string |
| `attachments` | optional array of metadata objects |
| `diagnostics` | optional object; unknown keys rejected or ignored per validator policy without dropping `message` |
| `diagnostics.platform` | `ios` \| `android` when present |
| `diagnostics.recent_error_codes` | array of strings |

**Responses:**

`201 Created` (or `200` on idempotent replay):

```json
{
  "ticket_id": "T-1042",
  "status": "NEW",
  "created_at": "2026-10-03T12:00:00Z"
}
```

Idempotent replay with same principal + key returns the original ticket payload (AC-002). Same key with a different body hash → `409` conflict.

### 2.2 Get ticket (mobile/test)

```http
GET /api/v1/tickets/{ticket_id}
Authorization: Bearer <token>
```

Returns customer-visible subset (status, subject, created_at). AI fields are agent-facing only for the POC unless explicitly productized later.

### 2.3 Attachment upload (optional)

```http
POST /api/v1/attachments/presign
Authorization: Bearer <mobile_jwt>
```

```json
{
  "filename": "screenshot.png",
  "content_type": "image/png",
  "size_bytes": 184220
}
```

Response includes presigned PUT URL and `storage_key`. Allowed types: image MIME allowlist; max size configurable (e.g. 5 MB).

---

## 3. Agent Ticket APIs

All routes require `role=agent` or `admin`.

### 3.1 Queue list

```http
GET /api/v1/agent/tickets
```

Query parameters:

| Param | Description |
|---|---|
| `status` | multi / comma-separated |
| `priority` | P0–P3 |
| `category` | category code |
| `team` | assigned team code |
| `needs_review` | `true` \| `false` |
| `q` | search ticket id or customer text |
| `sort` | `created_at` \| `priority` |
| `order` | `asc` \| `desc` |
| `limit` | default 50 |
| `cursor` | opaque pagination |

**Item shape:**

```json
{
  "ticket_id": "T-1042",
  "subject": "Premium not activated",
  "summary": "Premium purchase not reflected in app entitlements.",
  "priority": "P1",
  "category": "BILLING",
  "status": "NEEDS_REVIEW",
  "assigned_team": "billing_support",
  "recommended_team": "billing_support",
  "confidence": {
    "category": 0.71,
    "priority": 0.64,
    "routing": 0.77,
    "min": 0.64
  },
  "created_at": "2026-10-03T12:00:00Z"
}
```

### 3.2 Ticket detail

```http
GET /api/v1/agent/tickets/{ticket_id}
```

```json
{
  "ticket_id": "T-1042",
  "status": "NEEDS_REVIEW",
  "created_at": "2026-10-03T12:00:00Z",
  "customer": {
    "subject": "Premium not activated",
    "message": "I paid for premium but the app still says I am on the free plan.",
    "provenance": "customer"
  },
  "diagnostics": {
    "provenance": "diagnostics",
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
  },
  "ai": {
    "provenance": "ai",
    "summary": "Premium purchase not reflected in app entitlements.",
    "category": "BILLING",
    "subcategory": "subscription_activation",
    "priority": "P1",
    "routing": {"team": "billing_support"},
    "sentiment": "frustrated",
    "tags": ["premium", "entitlements"],
    "diagnosis": {
      "likely_cause": "Subscription sync failed after purchase",
      "confidence": 0.62
    },
    "suggested_actions": [
      "Verify store receipt against account",
      "Check subscription sync jobs for session-123"
    ],
    "suggested_reply": "Thanks for reporting this...",
    "confidence": {
      "category": 0.71,
      "priority": 0.64,
      "routing": 0.77
    },
    "model_metadata": {
      "provider": "openai_compatible",
      "model": "gpt-4.1-mini",
      "prompt_version": "triage-v1",
      "latency_ms": 1800,
      "estimated_cost_usd": 0.0021
    }
  },
  "decision": {
    "category": "BILLING",
    "subcategory": "subscription_activation",
    "priority": "P1",
    "priority_source": "ai",
    "assigned_team": "billing_support",
    "routing_source": "ai",
    "recommended_team": "billing_support",
    "tags": ["premium", "entitlements"],
    "suggested_reply": "Thanks for reporting this...",
    "classification_source": "ai"
  },
  "attachments": [],
  "reviewed_at": null
}
```

### 3.3 Approve AI triage

```http
POST /api/v1/agent/tickets/{ticket_id}/approve
```

```json
{
  "unchanged": true
}
```

Records approval signal (FR-FBK-003); sets `reviewed_at` if unset; may transition `NEEDS_REVIEW → OPEN` when appropriate.

### 3.4 Correct triage fields

```http
PATCH /api/v1/agent/tickets/{ticket_id}/decision
```

```json
{
  "category": "BILLING",
  "subcategory": "subscription",
  "priority": "P1",
  "assigned_team": "billing_support",
  "tags": ["premium", "entitlements", "sync"],
  "suggested_reply": "Edited reply..."
}
```

Server writes `feedback_corrections` for each changed field comparing against AI prediction; updates `ticket_decisions` with `*_source = human` where applicable; preserves `ai_triage_results` (AC-007).

### 3.5 Status transition

```http
POST /api/v1/agent/tickets/{ticket_id}/status
```

```json
{
  "status": "IN_PROGRESS"
}
```

Illegal transitions → `409` with machine-readable code. Always writes status audit (AC-010).

---

## 4. Admin APIs

Require `role=admin`. Agent credentials → `403` (AC-012).

### 4.1 Taxonomy

```http
GET    /api/v1/admin/taxonomy
POST   /api/v1/admin/taxonomy/categories
PATCH  /api/v1/admin/taxonomy/categories/{code}
POST   /api/v1/admin/taxonomy/categories/{code}/subcategories
PATCH  /api/v1/admin/taxonomy/subcategories/{category_code}/{sub_code}
```

Bodies include `name`, `is_active`. Deactivate does not delete historical references.

### 4.2 Teams

```http
GET    /api/v1/admin/teams
POST   /api/v1/admin/teams
PATCH  /api/v1/admin/teams/{code}
```

### 4.3 Rules

```http
GET    /api/v1/admin/rules
POST   /api/v1/admin/rules
PATCH  /api/v1/admin/rules/{id}
```

```json
{
  "name": "Subscription sync high priority",
  "is_active": true,
  "sort_order": 10,
  "match_predicate": {
    "all": [
      {"field": "diagnostics.recent_error_codes", "contains": "SUBSCRIPTION_SYNC_FAILED"}
    ]
  },
  "set_priority": "P1",
  "set_team_code": "billing_support"
}
```

### 4.4 Configuration / thresholds

```http
GET   /api/v1/admin/config
PATCH /api/v1/admin/config
```

```json
{
  "low_confidence_threshold": 0.55,
  "triage_max_retries": 3,
  "triage_timeout_seconds": 30,
  "ai_model": "gpt-4.1-mini",
  "prompt_version": "triage-v1",
  "data_retention_days": 180
}
```

---

## 5. Analytics and Evaluation Export

```http
GET /api/v1/analytics/overview
Authorization: Bearer <agent_or_admin_jwt>
```

```json
{
  "denominator": "reviewed_tickets_only",
  "reviewed_ticket_count": 120,
  "unreviewed_ticket_count": 40,
  "category_agreement": 0.83,
  "subcategory_agreement": 0.76,
  "priority_agreement": 0.79,
  "routing_agreement": 0.81,
  "override_rate": 0.22,
  "needs_review_rate": 0.18,
  "triage_latency_ms": {"p50": 2100, "p95": 7800},
  "triage_failure_rate": 0.03,
  "estimated_cost_usd_per_ticket": 0.0024
}
```

```http
GET /api/v1/analytics/evaluation.csv
```

CSV/JSON lines with: `ticket_id`, AI fields, final fields, correction flags, agent id, reviewed_at — suitable for offline evaluation (FR-FBK-004, AC-015). Unreviewed tickets excluded from agreement metrics; export may include them with `reviewed=false`.

---

## 6. Auth Helper (POC)

```http
POST /api/v1/auth/login
```

```json
{
  "email": "agent@example.com",
  "password": "..."
}
```

```json
{
  "access_token": "<jwt>",
  "token_type": "bearer",
  "role": "agent"
}
```

Mobile POC tokens may be issued by a test endpoint or seeded static JWT for demos; production IdP is out of scope.

---

## 7. AI Structured Output Schema (provider boundary)

Server asks the LLM for JSON conforming to this schema (FR-AI-003). Application validates with Pydantic before persistence (FR-AI-004, AC-004/005).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AiTriageOutput",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "summary",
    "category",
    "subcategory",
    "priority",
    "routing",
    "sentiment",
    "tags",
    "diagnosis",
    "suggested_actions",
    "suggested_reply",
    "confidence"
  ],
  "properties": {
    "summary": {"type": "string", "minLength": 1, "maxLength": 500},
    "category": {"type": "string", "minLength": 1},
    "subcategory": {"type": "string", "minLength": 1},
    "priority": {"type": "string", "enum": ["P0", "P1", "P2", "P3"]},
    "routing": {
      "type": "object",
      "additionalProperties": false,
      "required": ["team"],
      "properties": {
        "team": {"type": "string", "minLength": 1}
      }
    },
    "sentiment": {"type": "string"},
    "tags": {
      "type": "array",
      "items": {"type": "string", "maxLength": 40},
      "maxItems": 20
    },
    "diagnosis": {
      "type": "object",
      "additionalProperties": false,
      "required": ["likely_cause", "confidence"],
      "properties": {
        "likely_cause": {"type": ["string", "null"]},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
      }
    },
    "suggested_actions": {
      "type": "array",
      "items": {"type": "string"},
      "maxItems": 10
    },
    "suggested_reply": {"type": "string"},
    "confidence": {
      "type": "object",
      "additionalProperties": false,
      "required": ["category", "priority", "routing"],
      "properties": {
        "category": {"type": "number", "minimum": 0, "maximum": 1},
        "priority": {"type": "number", "minimum": 0, "maximum": 1},
        "routing": {"type": "number", "minimum": 0, "maximum": 1}
      }
    }
  }
}
```

### 7.1 Post-schema application validation

After JSON Schema/Pydantic parse, the worker must also ensure:

1. `category` ∈ active taxonomy codes.
2. `subcategory` ∈ active children of that category.
3. `routing.team` ∈ active team codes.
4. Tags normalized: trim, lower-case, dedupe, length limit (FR-AI-009).
5. Sentiment must not alone force P0/P1 (FR-PRI-004 / FR-AI-010).

Failure → retry policy or `NEEDS_REVIEW` (AC-005).

### 7.2 Prompt input (not an HTTP API)

Internal `TriagePromptInput` passed to `TriageLLMPort`:

- customer `subject`, `message`
- allowlisted diagnostics
- active taxonomy tree
- active teams
- applicable rule summaries (optional hints; rules still applied deterministically after)
- output schema / instructions
- prompt version id

Provider credentials never appear in mobile or browser clients (§4.3, constraint 8).

---

## 8. Web UI Routes (non-JSON)

Served by the same FastAPI app for the POC console:

| Route | Purpose |
|---|---|
| `GET /login` | Agent/admin login form |
| `GET /queue` | HTMX ticket queue |
| `GET /tickets/{ticket_id}` | Detail + review |
| `GET /admin` | Admin home |
| `GET /admin/taxonomy` | Taxonomy management |
| `GET /admin/teams` | Team management |
| `GET /admin/rules` | Rules management |
| `GET /admin/config` | Thresholds |
| `GET /analytics` | Simple metrics dashboard |

UI mutations call the JSON APIs above or sibling form endpoints that invoke the same application services (backend remains authoritative).

---

## 9. Contract Test Checklist

- [ ] Create ticket returns `ticket_id` without waiting for AI  
- [ ] Idempotency key replay safe  
- [ ] Create succeeds when LLM stub is down  
- [ ] Agent detail separates `customer`, `diagnostics`, `ai`, `decision`  
- [ ] Correction preserves AI payload  
- [ ] Admin routes reject agent token  
- [ ] Analytics denominators use reviewed tickets only  
- [ ] AI schema rejects invented taxonomy values at worker boundary  
