# Dev Team

This directory contains a multi-agent development team. Each agent has a specialized role in the development cycle. The team is language-agnostic — language-specific knowledge is delivered via loadable skills.

## How to use

**Always start with the Tech Lead.** Describe your requirement and let the Tech Lead orchestrate the rest.

```
/agent:tech-lead I need a user authentication module with JWT tokens
```

Do not invoke other agents directly unless you have a specific, scoped task for them.

## The Team

| Agent | Role |
|---|---|
| `tech-lead` | Entry point. Decomposes requirements, orchestrates the cycle, owns quality gates. |
| `architect` | Designs module boundaries, interfaces, deployment requirements, and patterns before any code is written. |
| `backend-engineer` | Implements the application from the Architect's spec. Ensures code is container-ready. Does not write Dockerfiles or pipelines. |
| `devops-engineer` | Containerizes the application (Dockerfile, docker-compose) and builds the Bitbucket deployment pipeline. Invoked after Backend Engineer work passes code review. |
| `code-reviewer` | Reviews diffs for correctness, code smell, SOLID violations, and spec deviation. Reviews both application and deployment code. |
| `qa-engineer` | Writes and runs tests. Verifies behavior against requirements, including Docker build and health check. |
| `security-reviewer` | Audits for OWASP vulnerabilities, hardcoded secrets, insecure patterns, and Docker/pipeline security. |
| `retrospective` | Observes the completed cycle. Identifies friction points, AI-collaboration failures, and agent instruction gaps. Writes improvement suggestions to `_retrospective/`. |

## Development Cycle

```
Requirement
    ↓
Tech Lead — decomposes, creates task breakdown, opens cycle log
    ↓
Architect — writes spec to _specs/ (interfaces, modules, deployment requirements)
    ↓
Backend Engineer — implements application, writes Container Interface summary to cycle log
    ↓
Code Reviewer — reviews application code
    ↓
Backend Engineer — applies fixes
    ↓
DevOps Engineer — writes Dockerfile, docker-compose, Bitbucket pipeline
    ↓
Code Reviewer — reviews deployment config
    ↓
QA Engineer — tests application behavior + Docker build + health check
    ↓
Security Reviewer — audits application + Docker + pipeline (invoked for auth, I/O, APIs, user input, always for deployment config)
    ↓
Tech Lead — sign-off or loops back
    ↓
Retrospective — presents findings, proposes improvements, applies approved changes
```

## Skills

### Mandatory Engineering Skills (`architecture/`)

Loaded by the Architect, Backend Engineer, and Code Reviewer on every task. Changing these files updates the behaviour of all agents that load them — no need to edit agent definitions.

- `.claude/skills/architecture/ddd-and-onion.md` — Domain-Driven Design building blocks and 4-layer Clean Architecture with dependency rules, Independent Definitions Principle, explicit mapping strategy
- `.claude/skills/architecture/loose-coupling.md` — Domain events for intra-context, integration events and event bus for inter-context, webhooks for external notifications, anti-corruption layers
- `.claude/skills/architecture/cqs-cqrs.md` — Command/query separation, handler structure, when to escalate to full CQRS with separate read/write stores
- `.claude/skills/architecture/compliance-scope.md` — How the Architect classifies existing compliance violations (fix in-cycle / escalate to human / defer), the escalation format, and the Compliance Scope section written into every spec

### Deployment Skills (`deployment/`)

Loaded by the DevOps Engineer on every task.

- `.claude/skills/deployment/docker-management.md` — Dockerfile authoring, multi-stage builds, docker-compose, image security
- `.claude/skills/deployment/bitbucket-pipelines.md` — Pipeline structure, branching strategy, Docker build/push, secrets, deployment

### Language Skills (`lang/`)

Loaded by the Backend Engineer (and QA Engineer for testing conventions) depending on the project's language. Add a new subfolder under `lang/` to add support for another language.

**Node.js / TypeScript (`lang/nodejs/`):**
- `.claude/skills/lang/nodejs/index.md` — entry point (skill name: `nodejs`); instructs which sub-skills to load, covers runtime conventions (async, error handling, container-readiness, DI)
- `.claude/skills/lang/nodejs/typescript-safety.md` — forbidden `any`, `unknown` with type guards, type assertion rules, running `tsc --noEmit`
- `.claude/skills/lang/nodejs/interface-naming.md` — `I` prefix only for polymorphic contracts; no prefix for DTOs, domain models, data structures
- `.claude/skills/lang/nodejs/testing-guidelines.md` — node:assert, Sinon, builders, randomUUID, no DI container in unit tests; loaded by QA Engineer
- `.claude/skills/lang/nodejs/import-path-conventions.md` — path aliases via `package.json` `imports` field, `.ts` extension; **only for new projects or projects with an existing `imports` field**

**Other languages (not yet created):**
- `.claude/skills/lang/python/` — Python
- `.claude/skills/lang/go/` — Go

## Architecture Specs

The Architect writes specs to a `_specs/` directory at the root of the target project. Specs are Markdown files named after the feature (e.g., `_specs/user-auth.md`). They contain the domain model, bounded context, commands & queries, events, module boundaries, and deployment requirements. The Backend Engineer reads these before writing any code.

## Retrospective Files

The Retrospective agent writes to `_retrospective/` in this `dev-team` directory:

- `_retrospective/findings.md` — append-only log of every cycle's observations, friction points, and what went well
- `_retrospective/improvement-backlog.md` — prioritized list of pending improvements to agent definitions, marked done when applied

These files are the team's institutional memory. The Retrospective agent presents findings and applies approved improvements interactively at the end of every cycle. Review the backlog periodically for any deferred items still worth addressing.
