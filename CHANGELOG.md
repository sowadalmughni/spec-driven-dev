# Changelog

All notable changes to `spec-driven-dev` are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] — 2026-09-10

### Added

- `SKILL.md` — Core skill definition with four-phase workflow (Scan, PRD, Feature Spec, Architecture, Atomic Prompts)
- `templates/prd-template.md` — Structured PRD with goals, user stories, acceptance criteria, non-goals, and open questions
- `templates/feature-spec-template.md` — Component scope document with responsibility matrix, data flow, security and FinOps checklists
- `templates/architecture-template.md` — Full technical architecture contract: module structure, data model, API contracts, end-to-end data flow, security architecture, FinOps analysis, CLAUDE.md injection block
- `reference/atomic-prompt-structure.md` — SKILL/DEPENDS/TASKS/VERIFY/PASS-FAIL/REPORT/COMMIT format for implementation prompts
- `reference/vibe-coding-failure-modes.md` — Six failure mode definitions with bash detection scripts and remediation register template
- `reference/claude-md-injection.md` — Architecture lock injection template for persistent CLAUDE.md constraints
- Three invocation modes: Standard, Continuation, Audit, Spec-only
- Audit Mode: generates remediation register before any fix work
- CLAUDE.md injection: locks architecture into future Claude Code sessions

### Design Decisions

- Architecture approval is a hard gate. No code generation without an approved architecture.
- Templates are strict, not flexible. Flexibility during specification leads to ambiguity during implementation.
- Audit and remediation are always separate phases. Inline fixes during audit are forbidden.
- Every implementation task must have a bash VERIFY command. If it cannot be verified with a command, it is not atomic enough.
- The skill generates CLAUDE.md constraints after architecture approval to prevent context-blind code generation in future sessions.
