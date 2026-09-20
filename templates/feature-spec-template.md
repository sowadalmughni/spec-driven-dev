# Feature Spec: [Feature Name]

> **Status:** DRAFT | APPROVED | SUPERSEDED
> **Version:** 1.0
> **PRD Reference:** `./docs/prd-[feature-name].md`
> **Architecture:** `./docs/architecture-[feature-name].md`
> **Date:**

---

## Purpose

_One paragraph. What is this feature responsible for? Where does its responsibility start and where does it end?_

---

## Component Map

_Every component (UI, API, service, worker, cron, etc.) that will be created or modified. One row per component. No implementation details here — just names and responsibilities._

| Component | Type | Responsibility | New / Modified |
|-----------|------|---------------|----------------|
|           | UI Component / API Route / Service / Worker / Schema |       |                |

---

## Responsibility Matrix

_For each component listed above, define what it DOES and what it explicitly does NOT do._

### [Component Name]

**Does:**
- Accepts [input type] and [what it does with it]
- Returns [output type]
- Validates [what] before [action]

**Does NOT:**
- Does not send emails or notifications (that is [other component]'s job)
- Does not modify [other table/module]
- Does not call [external service] directly

**Reason for boundary:** _Why is this the right cut point?_

---

## Data Flow

_Trace a single user action from UI to database and back. Every hop is named. No gaps allowed._

```
User Action
    ↓
[UI Component] — validates: [what]
    ↓
[API Route: METHOD /path] — auth check: [yes/no, where]
    ↓
[Service Layer: functionName()] — business logic: [what]
    ↓
[Data Layer: RepositoryName] — query: [SELECT/INSERT/UPDATE]
    ↓
[Database: table_name] — RLS: [enabled/disabled/policy name]
    ↑
[Response shape] — returns: [what fields]
    ↑
[UI State Update]
```

---

## Error Scenarios

_Every error path the user can encounter. Each has a defined handling strategy._

| Scenario | Where It Occurs | User-Facing Message | System Action |
|----------|----------------|--------------------|--------------:|
| Network timeout |           |                    |               |
| Auth failure |           |                    |               |
| Validation error |           |                    |               |
| Database constraint violation |           |                    |               |
| Third-party API failure |           |                    |               |

---

## Edge Cases

_List every "what if" that was surfaced during PRD review._

| Edge Case | Expected Behavior |
|-----------|-------------------|
|           |                   |

---

## Dependencies Between Components

_Which component must be built before another can start? This becomes the task ordering for atomic prompts._

```
[Component A] must complete before [Component B] can start
Reason: [why]

[Component C] can run in parallel with [Component D]
Reason: [they share no dependencies]
```

---

## Integration Points

_Every external system or internal module this feature touches. None may be added during implementation without a spec amendment._

| System | How Used | Data Sent | Data Received | Failure Handling |
|--------|---------|-----------|---------------|-----------------|
|        |         |           |               |                 |

---

## Security Checklist

_Completed before architecture phase. Each item is CONFIRMED or NOT APPLICABLE with a reason._

- [ ] **Authentication:** All routes require auth. Auth mechanism: [JWT / session / API key]
- [ ] **Authorization:** Role-based access defined. Roles: [list]
- [ ] **Row Level Security:** Every table accessed by this feature has RLS policy defined. Policy: [name]
- [ ] **Input validation:** All inputs validated at [API layer / service layer / both]
- [ ] **Secrets:** No credentials hardcoded. Storage: [env / secrets manager]
- [ ] **Error messages:** No stack traces or internal details exposed to the client
- [ ] **Rate limiting:** Applied at [route / service / both]
- [ ] **OWASP Top 10:** [Injection / Broken Auth / Sensitive Data / XSS / IDOR] risks reviewed

---

## FinOps Checklist

_Any "yes" requires an explicit mitigation in the Architecture Spec._

- [ ] Does this feature write to the database in a loop? — If yes: batch or bulk operation required
- [ ] Does this feature call an external API per record? — If yes: queue + worker pattern required
- [ ] Does this feature store user-uploaded content? — If yes: size limits, storage lifecycle, and CDN strategy required
- [ ] Does this feature run a background job? — If yes: idempotency, retry limits, and dead-letter queue required
- [ ] Does this feature involve unbounded queries? — If yes: pagination and index strategy required

---

## Approval

| Role | Decision | Notes | Date |
|------|----------|-------|------|
| Engineering lead | APPROVED / CHANGES REQUESTED |       |      |

---

_This Feature Spec is the scope contract for the implementation phase. Any component, integration, or behavior not listed here requires a spec amendment approved before implementation begins._
