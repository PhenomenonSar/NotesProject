<!--
SYNC IMPACT REPORT
==================
Version change: [unversioned] → 1.0.0
Modified principles: N/A (initial adoption)
Added sections:
  - Core Principles (I–V)
  - Technology Stack
  - Development Workflow
  - Governance
Templates updated:
  - .specify/templates/plan-template.md ✅ compatible (no changes required)
  - .specify/templates/spec-template.md ✅ compatible (no changes required)
  - .specify/templates/tasks-template.md ✅ compatible (no changes required)
  - .specify/templates/commands/ ✅ no command files found
Deferred TODOs: none
-->

# Notes Microservice Constitution

## Core Principles

### I. Python-First

The service MUST be implemented in Python (3.11+). All source code, scripts, and tooling
MUST use Python idioms and the standard library wherever sufficient. Third-party dependencies
MUST be explicitly justified — prefer well-maintained packages (e.g., FastAPI, SQLAlchemy,
Pydantic) over custom implementations of solved problems. No other language may be introduced
without a constitution amendment.

### II. REST API Design

Every feature MUST be exposed as a clean HTTP REST endpoint. Resources MUST be nouns
(e.g., `/notes`, `/notes/{id}`), HTTP verbs MUST express intent (GET, POST, PUT, DELETE).
Request and response bodies MUST use JSON. API contracts MUST be defined before implementation
begins — no undocumented endpoints may be merged.

### III. Atomic Commits (NON-NEGOTIABLE)

One task = one commit. Each commit MUST correspond to exactly one task from `tasks.md`.
Commits MUST NOT bundle multiple tasks, and tasks MUST NOT be split across multiple commits.
Commit messages MUST reference the task ID (e.g., `feat(T012): implement Note model`).
This rule is non-negotiable and supersedes convenience.

**Rationale**: Guarantees traceable, reviewable history where each increment is independently
deployable and revertable.

### IV. Simplicity (YAGNI)

Implementation complexity MUST be justified by current requirements, not anticipated future
needs. Abstractions MUST NOT be introduced unless used in at least two distinct places.
Premature optimization, unused configuration options, and speculative generalization are
prohibited. The simplest solution that satisfies acceptance criteria MUST be preferred.

### V. Observability

Every service operation MUST emit structured logs (key=value or JSON format) to stdout.
The service MUST expose a `GET /health` endpoint returning HTTP 200 with service status.
Errors MUST be logged with sufficient context to diagnose root cause without attaching a
debugger. Silent failures are prohibited.

## Technology Stack

- **Language**: Python 3.11+
- **Web Framework**: FastAPI (preferred) or Flask — MUST be declared in `requirements.txt`
  or `pyproject.toml`
- **Data Validation**: Pydantic v2
- **Storage**: Chosen per feature spec (SQLite for local/dev, PostgreSQL for production)
- **Testing**: pytest — unit and integration tests MUST pass before a task commit is valid
- **Linting/Formatting**: `ruff` for linting, `black` for formatting — CI MUST enforce both

## Development Workflow

- Tasks are sourced from `tasks.md` and executed in dependency order.
- Each task MUST be committed atomically (see Principle III).
- A task is considered complete only when: implementation is done, relevant tests pass,
  linting passes, and the commit is made with the correct task ID reference.
- No task may be marked complete without a corresponding commit.
- Branches MUST follow naming convention: `###-feature-name` (e.g., `001-notes-crud`).
- PRs MUST reference the feature spec and include a Constitution Check confirmation.

## Governance

This constitution supersedes all other development practices for this project. Amendments
require: (1) a written rationale, (2) version increment per semantic versioning rules,
(3) update of `LAST_AMENDED_DATE`, and (4) propagation review across all dependent templates.

All code reviews MUST verify compliance with Core Principles I–V. Violations block merge.
Complexity exceptions MUST be documented in the plan's Complexity Tracking table.

**Version**: 1.0.0 | **Ratified**: 2026-03-31 | **Last Amended**: 2026-03-31
