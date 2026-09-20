# CLAUDE.md Architecture Lock Injection

After the architecture spec is approved, this skill generates a block to inject into the project's `CLAUDE.md`. This block constrains every future Claude Code session to the approved architecture, preventing context-blind code generation from violating module boundaries.

## Why This Matters

Claude Code has no persistent memory between sessions. A developer who built the architecture in session 1 and returns for session 2 starts with a blank context. Without an explicit architecture lock in `CLAUDE.md`, the agent defaults to generating code based on what it sees in the current files — and it will guess at integrations, violate module boundaries, and generate schema changes without migrations.

The CLAUDE.md injection turns the architecture spec into a persistent constraint that loads at the start of every session.

---

## Injection Template

```markdown
## Architecture Lock: [Feature Name]

This project uses Spec-Driven Development. The architecture for [Feature Name] is approved and locked.
Spec: `./docs/architecture-[feature-name].md`

### MANDATORY CONSTRAINTS

Before writing any code in this codebase, read `./docs/architecture-[feature-name].md`.

**Module structure (do not deviate without a spec amendment):**
- [Domain] code lives in: `src/modules/[domain-name]/`
- Database changes: require migration file in `src/database/migrations/`
- Shared utilities: `src/shared/[utility-name]/`
- Dependency direction: modules import from shared/ only. Never import cross-domain.

**Database:**
- Schema: defined in `./docs/architecture-[feature-name].md § Data Model`
- All new columns require a migration file — no column additions via ORM sync
- Tables with RLS: [table_name_1], [table_name_2] — RLS must remain enabled
- Do not add columns to these tables without a schema amendment: [list]

**API contracts:**
- Endpoints: defined in `./docs/architecture-[feature-name].md § API Contracts`
- Response shapes are fixed — do not add or remove fields without a contract amendment
- Error codes follow the standard at `./docs/architecture-[feature-name].md § Error Responses`

**Authentication:**
- Auth mechanism: [JWT / session / API key]
- Every route requires `@UseGuards(JwtAuthGuard)` unless explicitly listed as public
- Public routes (no auth required): [list or "none"]
- Auth checks happen at: [guard layer] — never in the UI or repository layer

**Security:**
- RLS is active on: [list tables]
- Never disable RLS. Never run queries with service role key from client-side code.
- Input validation happens at: DTO layer (class-validator) — not inline in controllers

**External integrations:**
- [Service name]: called only through `[ServiceClassName]` — never direct fetch
- Credentials: stored in env as `[ENV_VAR_NAME]` — never hardcoded

**What requires a spec amendment before implementation:**
- New database tables or columns
- New API endpoints
- New external integrations
- Changes to authentication mechanism
- Changes to RLS policies
- New shared modules

### IMPLEMENTATION WORKFLOW

If asked to build, modify, or extend anything in this codebase:

1. Read the relevant section of `./docs/architecture-[feature-name].md`
2. Confirm the request is within approved scope
3. Use atomic prompt structure from `./skills/spec-driven-dev/reference/atomic-prompt-structure.md`
4. Include VERIFY bash commands for every task
5. Do not proceed without explicit pass criteria

### AUDIT MODE

If asked to review or audit this codebase:

1. Run the Six Failure Mode scan: `./skills/spec-driven-dev/reference/vibe-coding-failure-modes.md`
2. Output a remediation register before proposing any fixes
3. Do not fix inline during audit — audit and remediate are separate phases
```

---

## How to Inject

After the architecture spec is approved, run:

```bash
# Append the architecture lock block to the project's CLAUDE.md
cat ./skills/spec-driven-dev/reference/claude-md-injection.md >> ./CLAUDE.md
```

Or, if the project has no CLAUDE.md yet, this skill creates one:

```bash
# Generate CLAUDE.md from scratch with architecture lock
echo "# [Project Name] — Claude Code Context" > ./CLAUDE.md
echo "" >> ./CLAUDE.md
echo "Read this file at the start of every session before writing any code." >> ./CLAUDE.md
echo "" >> ./CLAUDE.md
# [Generated architecture lock block appended here]
```

---

## Maintenance

Update the CLAUDE.md injection block when:
- A new feature architecture is approved (append a new lock block)
- A schema amendment is approved (update the relevant constraint)
- A security change is approved (update the auth/RLS constraints)

Stale CLAUDE.md constraints are as dangerous as no constraints — a developer will follow outdated rules and generate code against an architecture that no longer exists.
