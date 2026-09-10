# Nexus AI Gateway Engineering Constitution

**Version:** 1.0.0

This document is the supreme engineering authority for the Nexus AI Gateway project. All code, architecture, and operational decisions must strictly adhere to the rules defined herein. No pull request may be merged, and no system deployed, if it violates this constitution.

## Mission

To build and maintain a reliable, high-throughput, and secure API gateway for routing, rate-limiting, and auditing Large Language Model (LLM) requests across multiple providers (OpenAI, Anthropic, local models) with built-in Retrieval-Augmented Generation (RAG) orchestration.

## Core Values

1. **Correctness over speed:** A working, accurate system is infinitely more valuable than a fast, broken one.
2. **Security over convenience:** Security is non-negotiable and must never be bypassed for developer velocity.
3. **Simplicity over cleverness:** Code must be readable and understandable by any engineer on the team.
4. **Maintainability over shortcuts:** Technical debt must be deliberate, documented, and short-lived.
5. **Observability over assumptions:** If we cannot measure it, we do not know if it works.
6. **Explicitness over magic:** Avoid implicit behaviors, hidden side-effects, and overly abstracted frameworks.
7. **Automation over manual processes:** If a task is done more than twice, it must be automated.
8. **Testing over trust:** All logic must be proven correct through automated tests.

## Technology Stack

### Required Technologies
* **Languages:** TypeScript (Node.js), Go
* **Frameworks:** Fastify (TypeScript), Fiber (Go)
* **Data Stores:** PostgreSQL (relational data, audit logs), Redis (caching, rate limiting)
* **Infrastructure:** Docker, Kubernetes, Helm
* **Validation:** Zod (TypeScript), `go-playground/validator` (Go)

### Forbidden Technologies / Practices
* Plain JavaScript in application code (TypeScript is mandatory).
* Unmaintained dependencies (libraries without updates in >12 months).
* Experimental libraries or beta framework features in production without explicit architecture board approval.
* ORMs that obscure raw SQL performance (prefer query builders like Kysely or raw SQL with `sqlx`).

## Repository Structure

The project utilizes a monorepo structure managed by Turborepo (for TS) and Go Workspaces.

```text
nexus-gateway/
├── apps/
│   ├── router-go/          # High-throughput Go routing engine
│   └── orchestrator-ts/    # TypeScript RAG and prompt orchestration
├── packages/
│   ├── core-types/         # Shared protobufs / type definitions
│   ├── db-client/          # Shared database connection and migration logic
│   └── llm-adapters/       # Provider-specific integration logic
├── tools/                  # Build scripts, CI/CD pipelines, linters
├── docs/                   # Architecture Decision Records (ADRs), runbooks
└── docker-compose.yml      # Local development environment
```

## Language/Code Standards

* **TypeScript:** Strict mode enabled (`"strict": true`). No `any` types allowed.
* **Go:** Must pass `gofmt`, `go vet`, and `golangci-lint` with all default linters enabled.
* **Naming Conventions:**
  * Variables, functions, methods: `camelCase`
  * Classes, Types, Interfaces, Structs: `PascalCase`
  * Database columns, JSON payloads: `snake_case`
  * File and directory names: `kebab-case`
* **Size Limits:**
  * Target file size: < 300 lines.
  * Mandatory refactor threshold: 500 lines.
  * Target function size: < 50 lines.

## Backend/API & Validation Standards

* **API Design:** All APIs must follow RESTful principles or use gRPC for internal service-to-service communication.
* **Versioning:** All public APIs must be versioned in the URL path (e.g., `/v1/chat/completions`).
* **Input Validation:** Every incoming request (headers, query parameters, body) must be strictly validated at the boundary using Zod (TS) or struct tags (Go). Unrecognized fields must be stripped or rejected.
* **Timeouts:** No network call may be made without an explicit timeout. Default LLM provider timeout is 30 seconds; default internal service timeout is 5 seconds.
* **Rate Limiting:** All endpoints must implement Redis-backed rate limiting based on the tenant ID or IP address.

## Error Handling

Errors must never expose internal stack traces or sensitive infrastructure details to the client. All errors must be mapped to one of the following standard categories:

| Error Category | HTTP Status | Description |
| :--- | :--- | :--- |
| `VALIDATION_ERROR` | 400 | Malformed request, missing fields, or invalid types. |
| `AUTHENTICATION_ERROR` | 401 | Missing, invalid, or expired credentials. |
| `AUTHORIZATION_ERROR` | 403 | Valid credentials, but insufficient permissions. |
| `BUSINESS_ERROR` | 422 | Request is valid but violates business rules (e.g., quota exceeded). |
| `EXTERNAL_SERVICE_ERROR` | 502 / 504 | Upstream LLM provider failed or timed out. |
| `INFRASTRUCTURE_ERROR` | 503 | Database, Redis, or internal network failure. |
| `UNKNOWN_ERROR` | 500 | Unhandled exceptions. Must trigger an immediate alert. |

## Logging

All logs must be structured JSON output to `stdout`. 

**Required structured log fields:**
* `event`: String describing the action (e.g., `llm_request_routed`).
* `timestamp`: ISO-8601 UTC timestamp.
* `requestId`: UUID tracing the request through the system.
* `userId` (optional): ID of the authenticated user/tenant.
* `metadata` (optional): Contextual data (e.g., `provider`, `model`, `token_count`).

*Note: PII, credentials, and raw LLM prompt/response text must NEVER be logged unless explicitly enabled for a specific tenant with legal consent.*

## Security

* **Authentication & Authorization:** Must occur entirely server-side. API keys must be hashed (SHA-256) in the database.
* **Secrets Handling:** 
  * Secrets must be injected via environment variables or a secret manager (e.g., AWS Secrets Manager, HashiCorp Vault).
  * Never commit secrets to version control.
  * Hardcoded credentials of any kind will result in immediate PR rejection.
* **Dependency Policy:**
  * Must pass automated security scans (e.g., Trivy, Snyk) before merge.
  * Must pass license review (GPL/AGPL licenses are strictly forbidden).
  * Must be actively maintained.
  * Prefer building over adding a dependency when the required functionality is small and well-understood.

## Performance

* **Latency:** Gateway routing overhead must not exceed 15ms at the 95th percentile (p95).
* **Throughput:** The system must support a minimum of 5,000 requests per second per node.
* **Caching:** Semantic caching and exact-match caching must be utilized via Redis to reduce upstream LLM calls where applicable.
* **Connections:** Database and Redis connection pooling must be explicitly configured and tuned for the deployment environment.

## Testing

* **Minimum Coverage:** 80% minimum overall coverage; 95% mandatory for critical business logic (routing, rate limiting, billing/token counting).
* **Required Test Types:**
  * **Unit Tests:** For all pure functions, parsers, and business logic.
  * **Integration Tests:** For database queries, Redis interactions, and API boundaries (using Testcontainers).
  * **Contract Tests:** To verify upstream LLM provider API compatibility.
  * **Load Tests:** k6 scripts must be maintained and run against staging before major releases.

## CI/CD

Every Pull Request must pass the following automated gates before it can be merged:
1. **Lint:** Code style and static analysis checks pass.
2. **Typecheck:** TypeScript compilation and Go build succeed without warnings.
3. **Unit Tests:** All unit tests pass and coverage thresholds are met.
4. **Integration Tests:** All integration tests pass in an ephemeral environment.
5. **Security Scan:** No high or critical vulnerabilities detected.

## Documentation

* **Architecture:** All significant architectural changes must be documented via Architecture Decision Records (ADRs) in `docs/adr/`.
* **API Documentation:** Must be automatically generated from code (e.g., OpenAPI/Swagger) and kept up to date.
* **Runbooks:** Every alert defined in the observability stack must have a corresponding runbook linked in the alert description.

## Observability

* **Metrics:** Prometheus metrics must be exposed on a dedicated internal port (e.g., `/metrics`). Required metrics include request counts, latency histograms, error rates, and token usage per provider.
* **Tracing:** OpenTelemetry (OTel) must be implemented to trace requests across the Go router, TS orchestrator, and external LLM providers.
* **Dashboards:** Grafana dashboards must be maintained as code (JSON) in the repository.

## AI Development Rules

* **AI-Generated Code Policy:** AI code is UNTRUSTED. It must be reviewed, tested, and validated by a human engineer before merging. The author assumes full responsibility for any AI-generated code they submit.
* **Agent Restrictions:** Autonomous agents (each without explicit human approval):
  * May NOT deploy to production.
  * May NOT rotate credentials or modify IAM policies.
  * May NOT modify infrastructure state (Terraform/Pulumi).
  * May NOT approve or merge pull requests.

## Prompt / MCP / RAG Standards

* **Prompt Governance:** 
  * System prompts and templates must be version-controlled, documented, and tested against a golden dataset.
  * Any changes to core prompts require a dedicated code review and regression testing.
* **MCP (Model Context Protocol) Integrations:** 
  * Must operate on a least-privilege basis.
  * Must be fully auditable (all tool calls logged).
  * Must be instantly revocable via feature flags.
* **RAG Standards:** 
  * Vector data sources must be trusted, versioned, and source-attributed in the final LLM response.
  * Chunking and embedding strategies must be documented in an ADR.

## Code Review Standards

Every Pull Request description must explicitly answer the following questions:
1. **What changed?** (Brief summary of the technical implementation)
2. **Why?** (Link to issue or business justification)
3. **Risks?** (What could break? Security implications?)
4. **Rollback plan?** (How do we revert if this fails in production?)
5. **Testing evidence?** (Screenshots, test output, or explanation of how it was verified)

## Git Standards

* **Branch Conventions:** `feature/*`, `bugfix/*`, `hotfix/*`, `chore/*`
* **Commit Conventions:** Conventional Commits are mandatory.
  * Types: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`, `chore`
  * Example: `feat(router): add rate limiting for anthropic endpoints`
* **History:** PRs must be squash-merged to `main` to maintain a linear, readable history.

## Dependency Rules

* Dependencies must be pinned to exact versions in lockfiles (`package-lock.json`, `go.sum`).
* Automated dependency updates (e.g., Dependabot, Renovate) are permitted but must pass all CI gates and require human approval before merge.
* Adding a new production dependency requires justification in the PR regarding its size, license, and security posture.

## Definition of Done

A feature or bugfix is only considered "Done" when:
- [ ] Requirements are fully implemented.
- [ ] Automated tests are written and passing.
- [ ] Typecheck and Linting are passing.
- [ ] Security review is completed (no new vulnerabilities introduced).
- [ ] Documentation (API docs, ADRs, Runbooks) is updated.
- [ ] Performance is validated (latency/throughput targets met).
- [ ] Code has been reviewed and approved by at least one peer.

*(Note: Accessibility validation is omitted as this is a backend API project. Assumption: No user-facing UI is served by this repository).*

## Non-Negotiable Rules (NEVER / ALWAYS)

* **NEVER** commit secrets, API keys, or credentials to version control.
* **NEVER** bypass CI/CD checks or force-push to `main`.
* **NEVER** log Personally Identifiable Information (PII) or raw LLM prompts without explicit, tenant-level configuration and legal consent.
* **NEVER** trust client input; always validate at the boundary.
* **ALWAYS** include a timeout for external network requests.
* **ALWAYS** write tests for new features or bug fixes.
* **ALWAYS** leave the codebase cleaner than you found it.

## Amendment Process

This constitution is a living document. To amend it:
1. Submit a written proposal via a Pull Request modifying this file.
2. The proposal must undergo an architecture review.
3. The proposal requires unanimous approval from the core engineering team.
4. Upon merge, the version number at the top of this document must be incremented (Semantic Versioning applies).