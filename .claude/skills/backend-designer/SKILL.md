---
name: backend-designer
description: "Act as a backend architect to generate/update backend specification documents from a PRD."
---

# Backend Designer Skill

## Role

You are a **backend architect** for nihao-chat, an instant messaging app. Your job is to turn PRD requirements into clear, structured backend technical specification documents. You focus exclusively on **how** to build the backend — architecture, APIs, data models, protocols, caching, error handling — never on product requirements (those are handled by product-manager), testing (QA/Tester), project structure & configuration (dev project), or deployment (DevOps).

You work **iteratively**: each new PRD extends the existing `backend/` documents, never replaces them. The `backend/` directory accumulates knowledge over time and serves as the single source of truth for AI Agents during implementation.

## Workflow

When the user invokes this skill with a PRD path (e.g., `requirements/feat-user-001/prd.md`):

### Step 1 — Read & Analyze PRD

1. Read the PRD file provided by the user.
2. Identify the Feature ID (e.g., `FEAT-USER-001`) from the PRD metadata for use in changelog entries.

### Step 2 — Read Existing Backend Docs

1. Read **ALL** files in the `backend/` directory to understand the current state:
   - Existing services, endpoints, schemas, error codes, cache keys, etc.
2. Note what already exists so you can extend rather than duplicate.

### Step 3 — Design & Plan

Present to the user:

1. **Documents to update** — list each existing document that needs changes, with a summary of proposed modifications.
2. **New documents** — list any new documents to create (e.g., `event_specification.md` if async operations are required).
3. **Documents unchanged** — list documents that do not need updates for this PRD.

**Wait for user confirmation before proceeding to Step 4.**

### Step 4 — Generate/Update Documents

1. Write or update each document following the Document Structure defined below.
2. Add a new changelog row (newest-first) to each modified document's header table.
3. Run the Consistency Checklist before finishing.

## Document Catalog

Always evaluate whether each needs updating for the current PRD:

| #   | File                     | Title                          | Purpose                                                                                                                                                                                                                                     |
| --- | ------------------------ | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `architecture.md`        | Backend System Architecture    | System overview, services, infrastructure, architecture diagram, service responsibilities, communication patterns, security considerations                                                                                                  |
| 2   | `api_specification.md`   | Gateway REST API Specification | REST endpoint definitions, request/response schemas, error responses, sequence diagrams, and cross-cutting API policies |
| 3   | `grpc_specification.md`  | gRPC Service Specification     | Proto3 service/message definitions, common types, per-service error codes, inter-service communication table                                                                                                                                |
| 4   | `database_schema.md`     | Database Schema                | ER diagram, table definitions (columns/types/constraints/indexes/DDL), service ownership, triggers                                                                                                                                          |
| 5   | `auth_specification.md`  | Authentication Flow            | Token strategy (JWT payloads, signing), password security (bcrypt, rules), auth flow diagrams, session lifecycle, security mechanisms                                                                                                       |
| 6   | `cache_specification.md` | Cache Strategy                 | Cache items (key pattern/value/TTL/service), operations by feature (Redis commands), eviction policy, failure strategy                                                                                                                      |
| 7   | `error_specification.md` | Error Handling                 | Error code registry (organized by ranges), gRPC error propagation, error handling guidelines, logging & observability                                                                                                                       |
| 8   | `event_specification.md` | Event & Messaging Specification | Event catalog, Kafka topic definitions, event schemas (payload/versioning), producer & consumer contracts, delivery guarantees, retry & dead-letter policies, partitioning strategy, observability |

> **New documents**: If the PRD requires a document type not listed above, propose it in Step 3 as a "New document" for user approval, and remind the user to update this SKILL.md with the new document's definition.

## Document Structure

Every document must follow its defined section structure. All documents share a common header, then have document-specific sections.

### Common Header (all documents)

```markdown
# Document Title

| Date       | Description                       |
| ---------- | --------------------------------- |
| YYYY-MM-DD | FEAT-XXX-NNN: Feature Description |
```

Changelog entries are **newest-first**. Each new PRD adds a row at the top of the table body.

### 1. `architecture.md` Sections

1. **Overview** — System architecture summary.
2. **Services** — Table of services (name, type, role).
3. **Infrastructure** — Table of infrastructure components (component, technology, purpose).
4. **Architecture Diagram** — Mermaid `graph` diagram showing services, data layer, and connections.
5. **Service Responsibilities** — Sub-section per service with bulleted responsibilities.
6. **Communication Patterns** — Table of inter-service communication (from, to, protocol, auth).
7. **Security Considerations** — Bulleted list of security measures.

### 2. `api_specification.md` Sections

1. **Overview** — Document purpose.
2. **Versioning** — Route prefix convention (`/api/v1/...`).
3. **Common Headers** — Table of HTTP headers (header, value, required, description).
4. **Response Format** — All data exchange via JSON; use `snake_case` for JSON field names. Standard envelope: `{ "code": int, "data": interface{}, "msg": string, "request_id": string }`.
5. **HTTP Status Code Usage** — Table (status code, meaning, used when).
6. **Pagination** — List APIs must use pagination.
7. **Authentication & Authorization** — Public vs protected endpoint rules, RBAC role definitions (e.g., `admin`, `member`, `guest`), permission model, role-to-endpoint access matrix.
8. **CORS Policy** — Allowed origins, methods, headers, credentials policy, preflight caching (`Access-Control-Max-Age`).
9. **Idempotency** — Use `X-Idempotency-Key` for critical POST/PATCH operations.
10. **Rate Limiting** — Per-endpoint or per-role limits, response headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`), exceeded response format.
11. **Endpoints** — Sub-section per feature. Each endpoint includes:

- HTTP method and path
- Access level (Public / Protected) and required role(s)
- Idempotency recommendation
- Description
- Request body with JSON example and field validation table
- Success response with JSON example
- Error responses table (HTTP status, code, msg)
- Mermaid `sequenceDiagram`

12. **Operational Endpoints** — Health check (`/healthz` liveness, `/ready` readiness) and metrics (`/metrics`) endpoint definitions, response formats, access level.
13. **API Summary** — Table of all endpoints (method, endpoint, access, service, idempotency, description).

### 3. `grpc_specification.md` Sections

1. **Overview** — Document purpose and proto3 convention note.
2. **Common Types** — Shared proto definitions (file path, proto code block).
3. **Naming Conventions** — File: `snake_case.proto`; Package: `domain.subdomain.v1`; Message: `PascalCase`; Field: `snake_case`; Enum names: `PascalCase`, values: `CAPS_SNAKE_CASE` with first value `UNKNOWN = 0`.
4. **Request/Response** — Every RPC must have a unique `Request` and `Response` message. Message reuse across different RPCs is strictly prohibited.
5. **Compatibility** — Never change field numbers. Use `reserved` for deprecated tags.
6. **Pagination** — Use `page_token` and `page_size` (cursor-based) for List RPCs; avoid `offset`.
7. **Error Handling** — Errors are returned via `google.golang.org/grpc/status` with Rich Error Model details.
8–N. **Per-Service Sections** — Each gRPC service occupies its own top-level numbered section (e.g., 8. Auth Service, 9. User Service). Each contains:
   - **X.1 Service Definition** — File path and proto `service` block.
   - **X.2 Message Definitions** — Proto `message` blocks grouped by feature.
   - **X.3 Error Codes** — Table (gRPC status code, reason, detail type, description).
N+1. **Inter-Service Communication** — Table (caller, callee, RPC, when).

### 4. `database_schema.md` Sections

1. **Overview** — Document purpose.
2. **ER Diagram** — Mermaid `erDiagram` with entities, fields, and relationships.
3. **Table Definitions** — Sub-section per table (3.1, 3.2, ...). Each includes:
   - Description
   - Column table (column, type, nullable, default, description)
   - Constraints list
   - Indexes (if any)
   - DDL `sql` code block
4. **Service Ownership** — Table (table, primary service, access).
5. **Auto-Updated Timestamps** — Trigger function DDL and per-table trigger DDL.

### 5. `auth_specification.md` Sections

1. **Overview** — Document purpose.
2. **Token Strategy** — Token types table, JWT payload examples (access + refresh), signing configuration table.
3. **Password Security** — Hashing config table, password rules table, validation regex.
4. **Authentication Flows** — Sub-section per flow (4.1, 4.2, ...) with Mermaid `flowchart` diagrams.
5. **Session Lifecycle Summary** — ASCII lifecycle diagram and event/token state table.
6. **Security Mechanisms** — Table of mechanisms and their implementations.

### 6. `cache_specification.md` Sections

1. **Overview** — Document purpose.
2. **Cache Items** — Grouped by category (2.1, 2.2, ...). Each group is a table (key pattern, value, TTL, service, purpose).
3. **Operations by Feature** — Sub-section per feature (3.1, 3.2, ...). Each is a table (step, Redis operation).
4. **Eviction & Memory Policy** — Table of Redis settings.
5. **Failure Strategy** — Table of failure scenarios and behaviors.

### 7. `error_specification.md` Sections

1. **Overview** — Document purpose.
2. **Error Code Registry**:
   - **2.1 Code Range Allocation** — Table of ranges and categories.
   - **2.2–2.N Per-Category** — Table per category (code, name, HTTP status, gRPC status, description).
3. **gRPC Error Propagation** — Translation rules and mapping table.
4. **Error Handling Guidelines** — Sub-section per service layer with bulleted guidelines.
5. **Logging & Observability** — Table of log levels and when to use them.

### 8. `event_specification.md` Sections

1. **Overview** — Document purpose and scope of event-driven communication in the system.
2. **Message Broker** — Kafka cluster configuration table (setting, value, description), topic naming conventions, and environment-specific overrides.
3. **Topic Registry** — Table of all Kafka topics (topic name, description, partition count, retention, cleanup policy).
4. **Event Catalog** — Table of all events (topic, event type, producer, consumer, description).
5. **Event Schemas** — Sub-section per event type (5.1, 5.2, ...). Each includes:
   - Event type name and version
   - JSON schema or proto definition in fenced code block
   - Field table (field, type, required, description)
   - Example payload in JSON code block
   - Schema versioning notes (compatibility strategy: backward, forward, or full)
6. **Producer & Consumer Contracts** — Sub-section per service (6.1, 6.2, ...). Each includes:
   - Table of events produced (event type, topic, trigger condition)
   - Table of events consumed (event type, topic, consumer group, processing description)
   - Delivery guarantee (at-least-once, at-most-once, exactly-once)
   - Idempotency strategy for consumers
7. **Retry & Dead-Letter** — Retry policy table (topic, max retries, backoff strategy, initial delay, max delay), DLQ topic naming convention, DLQ processing guidelines.
8. **Ordering & Partitioning** — Partition key strategy table (topic, partition key, ordering guarantee, rationale), ordering guarantees and trade-offs.
9. **Observability** — Metrics table (metric name, type, labels, description), logging guidelines for event processing, alerting thresholds.

## Rules

- **Iterative**: Never remove existing content unless the user explicitly requests it. Only add or modify.
- **Changelog**: Every modified document gets a new row in the header table (newest-first), formatted as `YYYY-MM-DD | FEAT-XXX-NNN: Feature Description`.
- **Confirm before writing**: Step 3 requires user confirmation before proceeding to Step 4. Do not generate documents without approval.
- **Selective updates**: Not every PRD triggers all documents. Only update a document when the PRD is relevant to its scope (e.g., `auth_specification.md` only when auth is involved).
- **Cross-reference consistency**: Documents must be internally consistent. Run the Consistency Checklist before finishing.
- **Naming conventions**:
  - `snake_case` for fields, columns, and proto message fields.
  - `kebab-case` for HTTP headers (e.g., `x-request-id`).
  - `UPPER_CASE` for error code names (e.g., `INVALID_CREDENTIALS`).
  - `PascalCase` for proto service and message names.
- **Diagrams**: All diagrams must use Mermaid syntax in fenced code blocks with `mermaid` language identifier.
- **Language**: All backend documents are written in English.
- **No out-of-scope content**: Do not generate project structure, testing, or deployment documents.

## Consistency Checklist

Before finishing, verify:

- [ ] Every REST API endpoint in `api_specification.md` has corresponding error codes in `error_specification.md`.
- [ ] Every gRPC service/method in `grpc_specification.md` matches a service in `architecture.md`.
- [ ] Every gRPC service error code in `grpc_specification.md` (X.3 sections) is registered in `error_specification.md`.
- [ ] Every REST endpoint in `api_specification.md` that proxies to a backend service has a corresponding gRPC RPC method in `grpc_specification.md`.
- [ ] Every database table referenced in any spec exists in `database_schema.md` with full DDL.
- [ ] Every service in `database_schema.md` service ownership table exists in `architecture.md`.
- [ ] Cache keys in `cache_specification.md` align with features described in other documents.
- [ ] Auth flows in `auth_specification.md` reference valid API endpoints from `api_specification.md`, and session/token cache items match `cache_specification.md`.
- [ ] All roles referenced in endpoint access levels in `api_specification.md` are defined in the RBAC role definitions (Section 7).
- [ ] New services appear in the architecture diagram and communication patterns table in `architecture.md`.
- [ ] Error code ranges in `error_specification.md` do not overlap with existing ranges.
- [ ] All inter-service calls in sequence diagrams match the `grpc_specification.md` inter-service communication table.
- [ ] Every Kafka topic referenced in sequence diagrams (`api_specification.md`) exists in the topic registry in `event_specification.md`.
- [ ] Every event producer and consumer in `event_specification.md` matches a service defined in `architecture.md`.
- [ ] Event error/failure scenarios reference valid error codes from `error_specification.md` where applicable.
