# Technical Architecture: [Feature Name]

> **Status:** DRAFT | APPROVED | SUPERSEDED
> **Version:** 1.0
> **Feature Spec:** `./docs/feature-spec-[feature-name].md`
> **PRD:** `./docs/prd-[feature-name].md`
> **Date:**

---

## Architecture Decision

_In two sentences: what architectural pattern was chosen and why. This justifies every structural decision below._

**Pattern chosen:** [Modular Monolith / Microservice / Serverless / etc.]

**Rationale:** _Why this pattern for this feature, given the constraints in the PRD._

---

## Module Structure

_Every file that will be created or modified. Exact paths. No "etc." or implied files._

> Shown below as a Node/NestJS/TypeORM reference example. Replace file names and layout with whatever matches this project's actual language, framework, and existing conventions — the requirement is explicit paths and clear module boundaries, not this specific framework.

```
src/
├── modules/
│   └── [domain-name]/              ← new domain module
│       ├── [domain-name].module.ts
│       ├── [domain-name].controller.ts
│       ├── [domain-name].service.ts
│       ├── [domain-name].repository.ts
│       ├── dto/
│       │   ├── create-[entity].dto.ts
│       │   └── update-[entity].dto.ts
│       └── entities/
│           └── [entity].entity.ts
├── database/
│   └── migrations/
│       └── [timestamp]-create-[entity].ts
└── shared/
    └── [shared-module-name]/       ← if shared utility needed
```

**Dependency rule:** Source code dependencies point inward only. `[domain-name]` module may import from `shared/` but may NOT import from other domain modules. Cross-domain communication goes through service interfaces.

---

## Data Model

_Complete schema. Every table, every column, every type, every constraint, every index. No columns added during implementation without a schema amendment._

> Shown below in Postgres SQL with Row Level Security as the reference example. If the project uses a different database or access-control model, translate the same requirements — explicit schema, explicit indexes, explicit row-level authorization — into that system's equivalent.

### Table: `[table_name]`

```sql
CREATE TABLE [table_name] (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  [column]    [TYPE] NOT NULL,                    -- reason for NOT NULL
  [column]    [TYPE],                             -- nullable: [why]
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes: every query pattern must have a supporting index
CREATE INDEX idx_[table]_[column] ON [table_name]([column]);
CREATE INDEX idx_[table]_user_id ON [table_name](user_id);  -- RLS pattern

-- Row Level Security
ALTER TABLE [table_name] ENABLE ROW LEVEL SECURITY;

CREATE POLICY "[table]_user_isolation" ON [table_name]
  FOR ALL
  USING (user_id = auth.uid());
```

**FinOps note:** _What is the expected row growth rate? Does this table need archival or partitioning?_

---

## API Contracts

_Every endpoint. Every method. Every request shape. Every response shape. Every error code. This is the contract the frontend and the backend both sign._

> Shown below in TypeScript/REST as the reference example. Adapt the shapes to the project's actual API style (REST, GraphQL, RPC) and language.

### `POST /api/[resource]`

**Auth required:** Yes — [Bearer JWT / API key / Session]
**Required role:** [role name or "authenticated user"]

**Request body:**
```typescript
{
  field: string;          // [validation: max 255 chars, required]
  optional_field?: number; // [validation: positive integer, defaults to 0]
}
```

**Success response (201):**
```typescript
{
  id: string;             // UUID
  field: string;
  created_at: string;     // ISO 8601
}
```

**Error responses:**
| Code | Condition | Body |
|------|-----------|------|
| 400 | Validation failure | `{ error: "VALIDATION_FAILED", fields: [...] }` |
| 401 | No auth token | `{ error: "UNAUTHORIZED" }` |
| 403 | Wrong role | `{ error: "FORBIDDEN" }` |
| 409 | Duplicate resource | `{ error: "CONFLICT", message: "..." }` |
| 500 | Unhandled server error | `{ error: "INTERNAL_ERROR" }` (no stack trace) |

**Rate limit:** [requests per minute / IP / user]

---

## End-to-End Data Flow

_The complete request lifecycle. Every hop is named. Every transformation is described. No implicit steps._

```
1. Client: [User action triggers HTTP POST /api/[resource]]
   Payload: { field: "value" }

2. API Gateway / Edge: [Auth token validated via middleware]
   Middleware: JwtAuthGuard → extracts user_id from JWT
   Passes: user_id to controller context

3. Controller: [ResourceController.create(dto, user)]
   Validates: class-validator on CreateResourceDto
   Calls: ResourceService.create(dto, user.id)

4. Service: [ResourceService.create(dto, userId)]
   Business logic: [describe what transforms happen]
   Calls: ResourceRepository.insert(entity)

5. Repository: [ResourceRepository.insert(entity)]
   Query: INSERT INTO [table_name] (user_id, field) VALUES ($1, $2) RETURNING *
   RLS active: user_id = auth.uid() policy enforced at DB layer

6. Database: [Postgres]
   Writes row. Returns inserted record.

7. Repository → Service → Controller:
   Maps DB record to response DTO

8. Client receives: { id, field, created_at }
```

---

## Integration Architecture

_Every external or internal system this feature calls. Connection method, authentication, error handling._

### [Integration Name: e.g., Stripe / SendGrid / internal Auth module]

**Connection method:** REST API / SDK / internal service call
**Authentication:** [API key in env / JWT / OAuth]
**Called from:** [Service layer only — never controller or repository]
**Timeout:** [ms]
**Retry strategy:** [exponential backoff, max 3 retries]
**Failure handling:** [queue for retry / fail fast / degrade gracefully]
**What we send:** `{ field: value }`
**What we receive:** `{ field: value }`
**Secrets stored in:** [environment variable name]

---

## Security Architecture

_Security decisions are made here, before code. Not during implementation, not "we'll add it later."_

### Authentication

- **Mechanism:** [JWT / session cookie / API key]
- **Token location:** [Authorization header / HttpOnly cookie]
- **Expiry:** [access token: 15min / refresh token: 7 days]
- **Renewal:** [refresh endpoint / silent renewal]

### Authorization

- **Model:** [RBAC / ABAC / simple auth check]
- **Roles involved:** [list]
- **Permission checks at:** [route guard → service layer — never UI-only]

### Row Level Security

- **Tables with RLS:** [list every table this feature touches]
- **Policy for each:** [policy name and rule]
- **Tables deliberately without RLS:** [none, or if yes: reason]

### Input Validation

- **Validation layer:** [DTO + class-validator at controller boundary]
- **Sanitization:** [DOMPurify for HTML / parameterized queries only — no string interpolation]
- **File uploads:** [max size: xMB / allowed types: [list] / stored in: S3 with random key]

### Secret Management

| Secret | Environment Variable | Scope |
|--------|---------------------|-------|
|        |                     |       |

---

## FinOps Architecture

_Any pattern that has infrastructure cost implications must be documented before implementation._

### Query Cost Analysis

| Query | Table | Estimated rows | Index used | N+1 risk | Mitigation |
|-------|-------|---------------|-----------|----------|------------|
|       |       |               |           | Yes / No |            |

### Storage Cost Analysis

| Data type | Growth rate | Retention policy | Estimated monthly cost |
|-----------|-------------|-----------------|----------------------|
|           |             |                 |                      |

### API Call Cost Analysis

| External API | Called per: request / user / event | Rate limit | Monthly est. at [X] users |
|--------------|-----------------------------------|-----------|---------------------------|
|              |                                   |           |                           |

---

## CLAUDE.md Injection Block

_Copy this block into the project's CLAUDE.md to lock architecture into future code generation sessions. This prevents future prompts from violating module boundaries._

```markdown
## Feature: [Feature Name] — Architecture Lock

This feature was built under a Spec-Driven Development workflow.
Approved architecture: `./docs/architecture-[feature-name].md`

CONSTRAINTS FOR ALL FUTURE CODE GENERATION:
- New files for [domain-name]: must go in `src/modules/[domain-name]/`
- Database changes: require migration file in `src/database/migrations/`
- Auth checks: implemented in JwtAuthGuard, not inline in controllers
- RLS: [table_name] has row-level security. Never disable it.
- External calls to [Integration]: only through [ServiceName], never direct
- Any deviation from the module structure requires architecture amendment
```

---

## Implementation Readiness Checklist

_Before atomic prompts begin, every item must be CONFIRMED._

- [ ] PRD approved: version [x.x], date [date]
- [ ] Feature Spec approved: version [x.x], date [date]
- [ ] All open questions from PRD are resolved
- [ ] Database migration written and reviewed
- [ ] RLS policies defined for every table
- [ ] All integration secrets added to environment
- [ ] No N+1 risk without a documented mitigation
- [ ] Error responses defined for every endpoint
- [ ] CLAUDE.md updated with architecture lock block

---

## Approval

| Role | Decision | Notes | Date |
|------|----------|-------|------|
| Engineering lead | APPROVED / CHANGES REQUESTED |       |      |
| Security review (if applicable) | APPROVED / SKIPPED — reason: |       |      |

---

_This architecture document is the implementation contract. Code that deviates from it without a formal amendment is out of scope and will not be merged._
