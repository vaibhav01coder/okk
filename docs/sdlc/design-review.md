# Design Review — KAN2

## Risks & Gaps Identified

### R-1 · Cache Invalidation Strategy Undefined
**Risk:** The architecture introduces Redis caching for read performance but specifies no invalidation or TTL strategy, creating a risk of stale `oojn`-scoped data being served after writes or external integration updates.
**Decision:** Define an explicit cache invalidation policy — either write-through invalidation triggered by Data Manager mutations, or a bounded TTL per `oojn` scope — and document it as a formal design decision before implementation begins.

### R-2 · External System (iuju) Availability Not Bounded by a Circuit Breaker SLA
**Risk:** The Integration Adapter references retry and circuit-breaker patterns in the tech stack but the architecture does not define open/half-open thresholds, timeout budgets, or a fallback response contract, meaning a degraded `iuju` will unpredictably consume the 1-second and 5-second response NFR budgets.
**Decision:** Specify concrete circuit-breaker parameters (failure-rate threshold, wait duration, timeout per call) and a documented fallback behaviour (e.g., serve partial cached data or return a structured degraded-mode response) as part of DD-03.

### R-3 · JWT Token Revocation Gap
**Risk:** Stateless JWT validation provides no mechanism to immediately revoke a compromised or role-changed token within its remaining TTL, violating the spirit of the RBAC NFR (NFR-05) and creating a privilege-persistence window.
**Decision:** Introduce a token denylist (Redis-backed, consistent with the existing cache infrastructure) or reduce JWT TTL to a short window (e.g., ≤5 minutes) paired with refresh-token rotation, and record this as a new design decision DD-08.

### R-4 · No Database Migration or Schema-Change Strategy
**Risk:** The architecture selects PostgreSQL and an ORM (Prisma/Hibernate) but does not address how schema migrations will be managed across environments, risking deployment failures and data integrity incidents during iterative development.
**Decision:** Mandate a versioned migration tool (e.g., Flyway or Prisma Migrate) with migration scripts committed alongside application code and executed as a pre-deploy step in the Kubernetes rollout pipeline; add this as DD-09.

### R-5 · Single Technology Choice Ambiguity (Node.js vs. Java Spring Boot)
**Risk:** Leaving the backend runtime undecided between Node.js/Express and Java Spring Boot defers a foundational choice that affects dependency selection (Zod vs. Bean Validation, Axios vs. Resilience4j), team skill requirements, and Docker image sizing, creating integration risk if the decision is made late.
**Decision:** Resolve the backend runtime choice as a gated decision before Sprint 1 API scaffolding; document the selected option with explicit rationale in DD-01 or a new DD entry, and remove the alternative from all subsequent artefacts.

### R-6 · Observability and Distributed Tracing Not Addressed
**Risk:** With multiple layers (Gateway → Auth → API → Data Manager → Integration Adapter → External), there is no mention of structured logging, metrics, or distributed trace correlation, making it impossible to diagnose latency budget breaches or cascading failures in production.
**Decision:** Mandate a minimum observability baseline — structured JSON logs with a correlation ID propagated from the API Gateway through all components, a metrics endpoint (e.g., Prometheus-compatible), and trace sampling via OpenTelemetry — and add this as DD-10 before the architecture is considered implementation-ready.

---

## Agreed Design Decisions

| ID | Decision |
|---|---|
| DD-01 | Separate RBAC middleware from the API service |
| DD-02 | Introduce a Redis cache layer for read performance |
| DD-03 | Use an Integration Adapter pattern for `iuju`, extended with defined circuit-breaker thresholds and a fallback response contract |
| DD-04 | Input/Output Processor as a dedicated component for `juuuh` I/O validation |
| DD-05 | Stateless API service instances behind the gateway for horizontal scaling |
| DD-06 | PostgreSQL as the primary data store |
| DD-07 | API Gateway rate limiting to protect backend under peak load |
| DD-08 | Redis-backed JWT denylist (or short-TTL + refresh-token rotation) to close the token revocation gap |
| DD-09 | Versioned database migration tooling (Flyway or Prisma Migrate) committed with application code and run pre-deploy |
| DD-10 | Mandatory observability baseline: structured logs with correlation ID, Prometheus metrics, and OpenTelemetry distributed tracing |

---

## Architecture Updates Applied

- **Cache Invalidation:** The Data Manager component description is updated to specify that all write operations trigger immediate cache key invalidation for the affected `oojn` scope; TTL is set as a secondary safety bound. This is reflected as an annotation on the `DM → CACHE` edge in the component diagram.
- **Circuit-Breaker Specification:** The Integration Adapter entry in the Components table is updated to include: timeout = 2 s per call, failure-rate threshold = 50 % over a 10-call sliding window, circuit open duration = 30 s, fallback = return last-known cached response or structured `503` with `Retry-After` header.
- **JWT Denylist:** A `TOKEN_DENYLIST` logical store (backed by the existing Redis instance) is added to the Data Layer. The Auth/RBAC Middleware data flow is updated to include a denylist lookup after signature validation and before forwarding to the API service.
- **Migration Pipeline:** The Infrastructure row in the Tech Stack table is updated to include Flyway (Java path) or Prisma Migrate (Node path) as a required pre-deploy Kubernetes init container step.
- **Backend Runtime Decision Gate:** A decision-gate milestone is inserted before Sprint 1 scaffolding, requiring sign-off on Node.js or Java Spring Boot; the architecture document will carry a single resolved entry upon completion of that gate.
- **Observability Layer:** An `OBS` cross-cutting component (OpenTelemetry Collector + Prometheus scrape endpoint) is added to the component diagram as a sidecar/agent consuming from all API Layer and Business Logic Layer components, with a note that correlation IDs are injected at the API Gateway and propagated in all inter-service headers.