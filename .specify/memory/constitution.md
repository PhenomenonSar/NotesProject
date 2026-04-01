<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 2.0.0
Modified principles:
  - I. Python-First → I. Java-First (language redefined: Python 3.11+ → Java 21 + Spring Boot 3.x)
  - V. Observability — unchanged in intent, updated stack references
Added sections: none
Removed sections: none
Changed sections:
  - Technology Stack: Python/FastAPI/pytest/ruff/black → Java/Spring Boot/Maven/no testing
  - Development Workflow: removed test-pass gate; no-test constitution
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ no changes required (placeholders are stack-agnostic)
  - .specify/templates/spec-template.md ✅ no changes required
  - .specify/templates/tasks-template.md ✅ tests already marked OPTIONAL
  - .specify/templates/commands/ ✅ no command files exist
Deferred TODOs: none
Version bump rationale: MAJOR — Principle I fully redefined (language change); testing requirement
  removed from workflow and technology stack. Backward-incompatible governance change.
-->

# Notes Microservice Constitution

## Core Principles

### I. Java-First

The service MUST be implemented in Java 21+. All source code, scripts, and tooling MUST use
Java idioms and the standard library wherever sufficient. Third-party dependencies MUST be
explicitly justified — prefer well-maintained, widely-adopted libraries (e.g., Spring Boot,
Spring Data JPA, Hibernate) over custom implementations of solved problems. No other language
may be introduced without a constitution amendment.

### II. REST API Design

Every feature MUST be exposed as a clean HTTP REST endpoint. Resources MUST be nouns
(e.g., `/notes`, `/notes/{id}`), HTTP verbs MUST express intent (GET, POST, PUT, DELETE).
Request and response bodies MUST use JSON. API contracts MUST be defined before implementation
begins — no undocumented endpoints may be merged.

### III. Atomic Commits (NON-NEGOTIABLE)

One task = one commit. Each commit MUST correspond to exactly one task from `tasks.md`.
Commits MUST NOT bundle multiple tasks, and tasks MUST NOT be split across multiple commits.
Commit messages MUST reference the task ID (e.g., `feat(T012): implement Note entity`).
This rule is non-negotiable and supersedes convenience.

**Rationale**: Guarantees traceable, reviewable history where each increment is independently
deployable and revertable.

### IV. Simplicity (YAGNI)

Implementation complexity MUST be justified by current requirements, not anticipated future
needs. Abstractions MUST NOT be introduced unless used in at least two distinct places.
Premature optimization, unused configuration options, and speculative generalization are
prohibited. The simplest solution that satisfies acceptance criteria MUST be preferred.

### V. Observability

Every service operation MUST emit structured logs to stdout via SLF4J/Logback.
The service MUST expose a `GET /actuator/health` endpoint (Spring Boot Actuator) returning
HTTP 200 with service status. Errors MUST be logged with sufficient context to diagnose root
cause without attaching a debugger. Silent failures are prohibited.

## Technology Stack

- **Language**: Java 21+
- **Framework**: Spring Boot 3.x (Spring Web MVC, Spring Data JPA)
- **Build Tool**: Maven (preferred) or Gradle — dependency manifest MUST be committed
- **Data Validation**: Bean Validation (Jakarta Validation API via Spring)
- **Storage**: Chosen per feature spec (H2 for local/dev, PostgreSQL for production)
- **Testing**: None — tests are explicitly out of scope for this project
- **Logging**: SLF4J + Logback (included with Spring Boot)
- **Health/Metrics**: Spring Boot Actuator

## Development Workflow

- Tasks are sourced from `tasks.md` and executed in dependency order.
- Each task MUST be committed atomically (see Principle III).
- A task is considered complete only when: implementation compiles, the application starts
  successfully, and the commit is made with the correct task ID reference.
- No task may be marked complete without a corresponding commit.
- Branches MUST follow naming convention: `###-feature-name` (e.g., `001-notes-crud`).
- PRs MUST reference the feature spec and include a Constitution Check confirmation.

## Governance

This constitution supersedes all other development practices for this project. Amendments
require: (1) a written rationale, (2) version increment per semantic versioning rules,
(3) update of `LAST_AMENDED_DATE`, and (4) propagation review across all dependent templates.

All code reviews MUST verify compliance with Core Principles I–V. Violations block merge.
Complexity exceptions MUST be documented in the plan's Complexity Tracking table.

**Version**: 2.0.0 | **Ratified**: 2026-03-31 | **Last Amended**: 2026-03-31
