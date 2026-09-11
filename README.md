# Dev Team

A multi-agent development team built with Claude Code. Each agent has a defined role in the development lifecycle. Together they carry a requirement from idea through architecture, implementation, containerization, testing, security review, and continuous process improvement.

The team is **language-agnostic**. Architecture concepts are universal. Language-specific knowledge (TypeScript conventions, testing patterns, import rules) is delivered via loadable skills — a new language is as simple as a new folder under `.claude/skills/lang/`.

---

## Purpose

This setup solves a specific problem: AI coding assistants are effective at individual tasks but tend to lose consistency, skip quality steps, or make assumptions without surfacing them when asked to handle an entire feature end-to-end. Each agent here has a narrow, well-defined scope. Handoffs are explicit. Quality gates are mandatory. The human stays in control of the decisions that matter.

---

## How to Use

**Always start with the Tech Lead.** Describe the requirement and let it orchestrate the rest.

```
/agent:tech-lead I need a user authentication module with JWT tokens
```

Do not invoke other agents directly for new features. The Tech Lead owns the cycle, routes work, holds the cycle log, and escalates to the human whenever a non-trivial decision arises.

---

## The Team

### Tech Lead
Entry point for every task. Receives requirements, creates a task breakdown, and orchestrates the full cycle. Owns all quality gates — no code proceeds to the next stage without sign-off. The only agent that talks to the human directly. Every other agent communicates through the Tech Lead.

**Communication topology:** hub-and-spoke. All agents route through the Tech Lead. No agent contacts another agent directly.

### Architect
Designs before any code is written. Produces a written spec in `_specs/<feature-name>.md` that defines the domain model, layer boundaries, events, commands and queries, and deployment requirements. On existing projects, runs a compliance gap assessment and escalates scope decisions to the human before designing — so the team never silently refactors code that wasn't part of the task.

### Backend Engineer
Implements the application from the Architect's spec. Responsible for domain logic, application layer, and ensuring the code can run inside a container (configurable port, health endpoint, stdout logging, graceful shutdown). Does not write Dockerfiles or pipelines.

### DevOps Engineer
Takes the running application and containerizes it. Writes the Dockerfile, docker-compose configuration, and Bitbucket pipeline. Reads the Backend Engineer's Container Interface summary from the cycle log before starting. Does not write application code.

### Code Reviewer
Reviews both the application code and the deployment configuration. Runs twice per cycle — once after the Backend Engineer, once after the DevOps Engineer. Reports findings by severity (Critical / High / Medium / Low). Critical and High findings loop back to the responsible engineer before the cycle continues.

### QA Engineer
Writes and runs tests against the spec and requirements — not against the implementation. Covers happy path, boundary values, error paths, and deployment verification (Docker build + health check). Uses the ubiquitous language from the spec in all test names.

### Security Reviewer
Audits for OWASP Top 10 vulnerabilities, hardcoded secrets, insecure dependencies, and unsafe patterns in both application code and deployment configuration. Invoked whenever the code touches auth, I/O, external APIs, database queries, or user input — and always for Dockerfiles and pipelines.

### Retrospective
Runs at the end of every cycle. Reads the cycle log and each agent's process notes to understand what happened inside the cycle, not just what was produced. Presents findings to the human, proposes concrete improvements to agent definitions, and applies the approved ones. This is how the team compounds improvement over time.

---

## Development Cycle

```
Requirement
    ↓
Tech Lead — decomposes, creates task breakdown, opens cycle log
    ↓
Architect — compliance assessment → spec to _specs/
    ↓
Backend Engineer — implements, writes Container Interface summary to cycle log
    ↓
Code Reviewer — reviews application code
    ↓
Backend Engineer — applies fixes
    ↓
DevOps Engineer — Dockerfile, docker-compose, Bitbucket pipeline
    ↓
Code Reviewer — reviews deployment config
    ↓
QA Engineer — application tests + Docker build + health check
    ↓
Security Reviewer — application + Docker + pipeline audit
    ↓
Tech Lead — sign-off or loops back
    ↓
Retrospective — interactive session with human: findings → proposals → approved changes applied
```

---

## Architecture Concepts

These concepts are mandatory on every feature. They are defined in skill files so they can be updated once and the change propagates to every agent that loads them.

### Domain-Driven Design (DDD)

The domain model is the center of the design. Code uses the language of the business — if domain experts call it an `Order`, the code has `Order`. The domain layer has zero external dependencies.

Key building blocks: **Aggregates** (consistency boundaries with a single root entry point), **Entities** (identity-based objects), **Value Objects** (immutable, attribute-based equality), **Domain Events** (immutable past-tense records of state changes), **Repositories** (collection-like persistence interfaces), **Factories** (complex object creation), **Domain Services** (stateless cross-aggregate operations).

Aggregates are encapsulated: private constructors, static `create()` factory methods, no `toJSON()`/`toDTO()` methods on domain objects — serialization is a mapper's job.

### Clean Architecture (4 Layers)

Dependencies point inward only. Each layer depends only on the layers inside it, never outward.

```
Interface / Presentation   ←  outermost: controllers, routes, validators
Infrastructure             ←  implements ports: databases, queues, external APIs
Application                ←  orchestrates use cases: DTOs, mappers, command/query handlers
Domain                     ←  innermost: pure business logic, no external dependencies
```

**Independent Definitions Principle:** each layer owns its own type definitions. Domain objects are never re-exported as DTOs. Explicit mapper classes transform data at every boundary.

### Loose Coupling

Components communicate via events, not direct calls:
- **Intra-context:** domain events (aggregate raises, application layer dispatches)
- **Inter-context:** integration events on an event bus
- **External notifications:** webhooks
- **Unavoidable sync calls:** anti-corruption layers behind a domain interface

### Command Query Separation (CQS)

Every operation is either a command (changes state, returns nothing) or a query (reads state, has no side effects). Never both. Separate `commands/` and `queries/` directories in the application layer make the separation visible.

### Compliance Scope Assessment

When working on existing projects that don't fully comply with the architecture concepts, the Architect classifies each violation found: fix in this cycle (code being modified anyway), escalate to human (requires touching code outside the feature boundary), or defer to the backlog. The human decides on escalated items before the spec is written. The Code Reviewer respects these decisions — approved deviations are not findings.

---

## Skills

Skills are Markdown files loaded by agents using the `Skill` tool. They define reference material and rules the agent applies in its role. Editing a skill file updates the behavior of every agent that loads it — no need to edit agent definitions.

```
.claude/skills/
├── architecture/            # Mandatory on every feature
│   ├── ddd-and-onion.md     # DDD building blocks + 4-layer Clean Architecture
│   ├── loose-coupling.md    # Event-driven communication patterns
│   ├── cqs-cqrs.md          # Command/query separation and CQRS
│   └── compliance-scope.md  # Compliance gap assessment for existing projects
├── deployment/              # Loaded by DevOps Engineer
│   ├── docker-management.md
│   └── bitbucket-pipelines.md
└── lang/                    # Language-specific skills
    └── nodejs/              # Node.js / TypeScript
        ├── index.md                  # Entry point (skill name: nodejs)
        ├── typescript-safety.md      # No any, unknown + type guards, tsc --noEmit
        ├── interface-naming.md       # I prefix rules for interfaces
        ├── testing-guidelines.md     # node:assert, Sinon, builders, test structure
        └── import-path-conventions.md # Path aliases (new projects or existing imports field only)
```

**To add a language:** create a subfolder under `lang/` with an `index.md` that has a unique `name:` in its frontmatter. Add sub-skills as needed. Reference the new skill name in the Backend Engineer's and QA Engineer's instructions.

---

## Observability and Improvement

### Cycle Log

The Tech Lead creates `_retrospective/cycle-<feature>-<YYYY-MM-DD>.md` at the start of every cycle. Every agent appends a process note when done — their honest account of input quality, decisions made, uncertainties, and confidence level. This gives the Retrospective agent visibility into what happened inside the cycle, not just what was produced.

### Retrospective Files

- `_retrospective/findings.md` — append-only log of every cycle's observations, friction points, and what went well. The team's institutional memory.
- `_retrospective/improvement-backlog.md` — pending improvements to agent definitions, marked done when applied.

Review the backlog periodically and apply items the Retrospective deferred. Each applied improvement is a permanent change to the agent files — the team gets better over time.

---

## Constraints and Design Decisions

**Agents do not contact each other directly.** All routing goes through the Tech Lead. This prevents cascade assumptions and keeps context centralized.

**Every cycle ends with a Retrospective.** Even a failed or abandoned cycle. The value is in observing what went wrong, not just shipping.

**Never assume on non-trivial questions.** The Tech Lead escalates to the human rather than letting an agent fill in a business or architectural gap. A wrong assumption discovered in QA costs more than pausing to ask.

**Compliance improvements are opt-in per cycle.** The team never silently refactors code outside the feature boundary. Scope decisions are human-approved before work begins.

**Skills over agent definitions for shared rules.** If a principle applies to multiple agents, it lives in a skill file — not duplicated across agent definitions. Agents load it. One edit propagates everywhere.

---

## File Structure

```
dev-team/
├── README.md                    # This file
├── CLAUDE.md                    # Project instructions loaded automatically by Claude Code
├── .claude/
│   ├── agents/                  # Agent definition files
│   │   ├── tech-lead.md
│   │   ├── architect.md
│   │   ├── backend-engineer.md
│   │   ├── devops-engineer.md
│   │   ├── code-reviewer.md
│   │   ├── qa-engineer.md
│   │   ├── security-reviewer.md
│   │   └── retrospective.md
│   └── skills/                  # Loadable skill files
│       ├── architecture/
│       ├── deployment/
│       └── lang/
├── _specs/                      # Architect writes specs here (created per project)
└── _retrospective/              # Cycle logs, findings, and improvement backlog
```
