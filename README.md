# spec-driven-dev

A Claude Code skill that enforces architecture-first development before generating any implementation code.

**The problem it solves:** AI tools generate code faster than any human can type. But generating code is not the same as building software. A founder can spend days building a product with AI assistance and end up with:

- A database schema that is never connected to the application
- Frontend pages that are fully styled but call no API
- Backend modules that exist but have no entry points
- Integrations that are mocked rather than wired
- No cohesive data flow through the entire system

This skill prevents that. It halts code generation until a Product Requirements Document and Technical Architecture Spec exist and are approved. Every implementation task is then generated as an atomic prompt with explicit file paths, numbered tasks, and bash VERIFY commands.

---

## Install

```bash
npx skills add https://github.com/sowadalmughni/spec-driven-dev
```

Or add to a project directly:

```bash
mkdir -p .claude/skills
git clone https://github.com/sowadalmughni/spec-driven-dev .claude/skills/spec-driven-dev
```

---

## How It Works

The skill activates when you ask Claude Code to build, implement, generate, create, or write anything. Instead of producing code, it runs a four-phase workflow:

**Phase 0 — Scan.** Checks whether a spec already exists for the requested feature. If yes, resumes from the approved spec. If no, starts Phase 1.

**Phase 1 — PRD.** Generates a Product Requirements Document using the included template. Asks for approval before continuing. Does not write code.

**Phase 2 — Feature Spec.** Generates a component breakdown that bridges the PRD to the technical design. Names every component, its responsibility, and its explicit non-responsibilities. Asks for approval before continuing.

**Phase 3 — Architecture.** Generates a full Technical Architecture Spec: module structure, database schema, API contracts, end-to-end data flow, security design, and FinOps risk analysis. Injects an architecture lock block into your project's `CLAUDE.md` so future sessions stay within the approved boundaries. Asks for approval before continuing.

**Phase 4 — Atomic Prompts.** Only after architecture approval does code generation begin. Every task references an exact file path, has numbered responsibilities, and includes bash VERIFY commands that confirm success.

---

## What Is Included

```
spec-driven-dev/
├── SKILL.md                              ← Skill definition and workflow rules
├── README.md                             ← This file
├── CHANGELOG.md
└── templates/
│   ├── prd-template.md                   ← Product Requirements Document structure
│   ├── feature-spec-template.md          ← Component scope and boundary document
│   └── architecture-template.md         ← Full technical design contract
└── reference/
    ├── atomic-prompt-structure.md        ← SKILL/DEPENDS/TASKS/VERIFY format
    ├── vibe-coding-failure-modes.md      ← Six failure mode detection scripts
    └── claude-md-injection.md            ← Architecture lock for CLAUDE.md
```

---

## The Six Failure Modes This Prevents

The `reference/vibe-coding-failure-modes.md` file contains bash detection scripts for each:

| # | Failure Mode | Detection Method |
|---|-------------|-----------------|
| 1 | Disconnected schema | Diff migration tables against repository queries |
| 2 | Unwired frontend | Grep for hardcoded mock data and missing API calls |
| 3 | Incomplete backend wiring | Find services never registered in modules |
| 4 | Missing Row Level Security | Check `pg_tables.rowsecurity` for every table |
| 5 | N+1 query explosions | Find query-in-loop patterns in ORM code |
| 6 | Exposed secrets and broken auth | Scan for credentials, check for bypassed auth guards |

Run these scripts directly in Audit Mode to generate a remediation register before any fix work begins.

---

## The Atomic Prompt Format

Generated implementation prompts follow this structure:

```
## SKILL
One sentence describing this task, traceable to a feature spec component.

## DEPENDS
Which previous tasks must complete first, and why.

## TASKS
1. Create ./src/[exact/path/to/file.ts]
   - [Specific responsibilities]

## VERIFY
```bash
test -f ./src/[path] && echo "PASS" || echo "FAIL"
npx tsc --noEmit && echo "PASS: compiles" || echo "FAIL"
```

## PASS-FAIL
Binary criteria. If a VERIFY command fails, implementation stops.

## COMMIT
git commit -m "feat: [feature] — [task description]"
```

Every prompt has one responsibility. Every prompt can be verified with a bash command. Every commit message is traceable to a feature spec.

---

## Methodology

This skill packages the documentation-first engineering workflow developed across 20+ AI-assisted product builds at [Kitalon Labs](https://www.kitalonlabs.com). The methodology emerged from building production SaaS applications with AI coding tools and discovering the specific failure modes that kill AI-assisted projects at month three — when the initial prototype is feature-complete on the surface but architecturally incoherent underneath.

The core thesis: AI tools are exceptional at implementation. They are blind to architecture. The engineer's job is to define the architecture with such precision that every implementation step is unambiguous. This skill enforces that precision.

---

## License

MIT — use freely, attribution appreciated.
