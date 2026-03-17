<!--
SYNC IMPACT REPORT
------------------
Version change: 1.0.0 -> 1.1.0
Modified principles: Purpose, Article XIV
Added sections: IaC mandate in Purpose and Article XIV
Removed sections: N/A
Templates requiring updates:
  - plan-template.md: Constitution Check section requires alignment with new IaC rules
  - spec-template.md: Already aligned with constitutional requirements
  - tasks-template.md: Already aligned with testing requirements
Follow-up TODOs: None
-->

# AdmissionZen Constitution

## Purpose

AdmissionZen is a multi-tenant microservices backend for students and families applying to universities in Canada, the United States, and the United Kingdom.

This repository governs the backend platform only. The system must support:
- student self-signup
- parent signup
- parent-created student profiles
- student invitation and account linking
- family management with one or more parents and one or more children
- student-scoped data isolation inside a shared family context
- future payment and entitlement management
- secure, observable, scalable service-to-service communication
- cross-platform search, analytics, and derived datasets for internal product and marketing use

This codebase is a shared backend monorepo containing multiple services and shared platform tooling.

The backend runtime target is AWS Fargate. Every deployable backend service must be containerized and compatible with stateless container execution on AWS Fargate.
Infrastructure and deployment resources MUST be managed as code.
AWS CDK is the default Infrastructure as Code standard for provisioning and managing the platform's AWS resources.

All specs, plans, and implementations MUST comply with this constitution.

---

## Article I — Domain-First Service Design

1. The platform MUST be designed around business domains, not around shared tables or convenience endpoints.
2. Each deployable service MUST own its own domain logic, API contract, and persistence boundary.
3. No service may directly write into another service's database or storage.
4. Cross-service coordination MUST happen through explicit APIs, asynchronous events, or approved integration contracts.
5. Shared database patterns between services are prohibited except for narrowly approved infrastructure concerns.
6. A shared monorepo does not justify runtime coupling or data-boundary violations.
7. New services MUST NOT be created unless there is a clear domain and operational reason.

**Rationale**: The repo is shared, but services must remain independently evolvable and independently deployable. Microservice count must remain intentional and controlled.

---

## Article II — Required Bounded Contexts

1. Every feature MUST belong to a named bounded context before implementation begins.
2. Initial bounded contexts SHOULD include at minimum:
   - Identity and Access
   - Family and Relationships
   - Student Profile
   - Applications and Admissions Workflow
   - Documents and Media
   - Search and Analytics
   - Notifications
   - Billing and Entitlements
   - Audit and Compliance
3. Bounded contexts may be combined into a smaller number of deployable services in early stages when this reduces operational complexity without violating ownership boundaries.
4. Each bounded context MUST define:
   - owned entities
   - owned API surface
   - emitted events
   - consumed events
   - source of truth
   - retention expectations
5. Ownership ambiguity is a design failure and MUST be resolved before coding.

**Rationale**: Bounded contexts define ownership. Deployable service count may evolve over time.

---

## Article III — Identity, Authentication, and Authorization

1. Auth0 is the system of record for authentication.
2. AdmissionZen services are the system of record for authorization.
3. Authentication and authorization MUST NOT be conflated.
4. Edge routing may be centralized, but authorization decisions MUST remain in the owning backend services.
5. Each service MUST validate Auth0 JWTs locally using cached signing keys or equivalent low-latency validation mechanisms.
6. Services MUST NOT require a network roundtrip to a centralized auth service for every request.
7. Authorization MUST be based on:
   - actor identity
   - family membership
   - actor role
   - student relationship context
8. Every access to student-scoped data MUST be attributable to a specific authenticated actor.

**Rationale**: This minimizes latency and avoids a central bottleneck while preserving domain-correct authorization.

---

## Article IV — Family Model Is Explicit

1. A student belongs to exactly one family.
2. A family may contain one or more parents or guardians and one or more children or students.
3. Parents associated with the family have full access to family and student data, equivalent to the student's access unless future product rules explicitly change this.
4. Family relationships MUST be modeled explicitly in domain data, not inferred from email addresses or metadata.
5. Student data MUST remain logically scoped to the student, even when accessible by multiple actors in the same family.
6. All family membership changes MUST be auditable.

---

## Article V — Parent-Created Student Profiles and Student Claiming

1. A parent may create a student profile before the student has a login identity.
2. A parent-created student profile is a domain entity, not automatically a login account.
3. The system MUST support later invitation, claiming, or linking of that student profile to the student's own Auth0 identity.
4. Claiming or linking a student profile MUST preserve the existing family, student history, and associated application data.
5. Ownership transfer of raw credentials is prohibited; identity linkage is the approved model.
6. Invitation and linking workflows MUST be secure, auditable, and idempotent.

**Rationale**: This avoids ambiguous credential ownership and keeps the family model stable.

---

## Article VI — API and Communication Standards

1. Every service MUST expose an explicit API contract before implementation.
2. REST is the default transport for synchronous APIs.
3. WebSocket is allowed only when real-time bidirectional behavior is genuinely required.
4. Webhooks MUST be idempotent, signed where applicable, replay-safe, and observable.
5. Breaking API changes are prohibited without versioning and migration planning.
6. Long synchronous dependency chains across services are prohibited.
7. Synchronous calls should be reserved for request/response needs where immediate consistency is required.

---

## Article VII — AWS-Native Messaging Standard

1. Cross-service asynchronous communication MUST prefer AWS-native messaging patterns.
2. The default event-driven stack SHOULD use:
   - EventBridge for domain event routing where fan-out and loose coupling are desired
   - SQS for durable queue-based work processing and retry control
   - SNS only where it is the clearest fit and does not duplicate EventBridge responsibilities
3. Internal HTTP APIs are acceptable for synchronous service-to-service interactions when justified.
4. Each asynchronous workflow MUST define:
   - producer
   - consumer
   - delivery expectations
   - retry behavior
   - idempotency strategy
   - dead-letter handling
5. Systems MUST assume duplicate delivery, retry storms, out-of-order events, and partial failure.

**Rationale**: AWS-native primitives align with Fargate deployment and reduce unnecessary infrastructure overhead.

---

## Article VIII — Events, Projections, and Data Ownership

1. Events are the preferred mechanism for propagating domain changes across services.
2. Every event MUST have:
   - a clear producer
   - a documented schema
   - a version
   - an idempotency strategy
   - a semantic meaning as a domain fact
3. Events SHOULD describe facts, not hidden RPC commands, unless command semantics are explicitly intended and documented.
4. Services that need read-optimized, analytics-optimized, or search-optimized data MUST maintain their own projections.
5. No service may directly read another service's operational tables as its integration strategy.
6. Cross-service materialized views must be built from events or approved ingestion flows.

---

## Article IX — Search and Analytics Owns Its Own Derived Data Layer

1. Search and Analytics is a dedicated bounded context with its own storage and indexing model.
2. Search and Analytics MUST NOT depend on direct reads from other services' operational databases.
3. Search indexes, analytical projections, and campaign-ready derived datasets MUST be built from domain events or approved ingestion contracts.
4. The architecture MUST support future platform-wide querying across:
   - students
   - families
   - applications
   - tasks
   - deadlines
   - notes
   - files
   - events
   - product engagement metrics
   - other approved indexed entities
5. Search and Analytics MAY support internal segmentation, dataset generation, key-metric exploration, and marketing campaign preparation, provided governance and privacy rules are enforced.
6. Derived analytics datasets MUST be documented with:
   - source systems
   - transformation logic
   - refresh strategy
   - retention policy
   - access policy
7. Search relevance, indexing strategy, and analytical models may evolve independently of source services, as long as ownership boundaries remain intact.

**Rationale**: Search is both a product capability and a derived data capability, but it must remain projection-based rather than bypassing service boundaries.

---

## Article X — Persistence and Storage Boundaries

1. Supabase Postgres is the primary relational datastore for structured backend data.
2. S3 is the primary object store for raw files, bulk media, exports, and large unstructured assets.
3. Cache usage MUST be explicit and safe to rebuild from source-of-truth systems.
4. Each service MUST document:
   - its source of truth
   - its replication or projection strategy
   - backup expectations
   - retention policy
5. Large binary content MUST NOT be stored in Postgres except for narrowly justified metadata-sized cases.
6. Direct cross-service writes and direct cross-service table coupling are prohibited.

---

## Article XI — Billing and Family Entitlements

1. Stripe is the approved billing platform for future payment workflows.
2. Billing and entitlements MUST be isolated in a dedicated bounded context or service.
3. Entitlements MUST be modeled at the family level.
4. The system MUST support pooled family entitlements, including limits based on the number of supported applications across the family.
5. Initial product assumptions should support:
   - free tier with limited family application capacity
   - paid tier with expanded or unlimited family application capacity
6. Entitlement enforcement MUST NOT be duplicated inconsistently across unrelated services.
7. Billing webhooks MUST be treated as external events and must be idempotent, auditable, and replay-safe.

**Rationale**: Your monetization is based on pooled application capacity, not simply per-user subscriptions.

---

## Article XII — Testing Is Mandatory

1. Unit tests are mandatory for all domain logic.
2. All new features MUST include tests for happy paths, edge cases, and failure paths.
3. Integration tests are required for:
   - API contracts
   - persistence behavior
   - authorization boundaries
   - event publication
   - event consumption
4. Contract testing is required for inter-service APIs and event schemas.
5. End-to-end tests SHOULD cover critical family journeys, including:
   - student signup
   - parent signup
   - parent-created student profile
   - student invitation and linking
   - multi-child family flows
   - entitlement enforcement
6. Code without meaningful automated tests is incomplete.
7. Mock-only tests are insufficient without business-behavior validation.

---

## Article XIII — Python and FastAPI Standards

1. Services MUST be implemented in Python using FastAPI unless an exception is explicitly approved.
2. Business logic MUST NOT live in controllers, routers, or transport handlers.
3. Code structure MUST separate:
   - transport layer
   - application or service layer
   - domain layer
   - infrastructure layer
4. Domain models, API DTOs, and persistence models MUST be treated as separate concerns unless explicitly justified.
5. Type hints are mandatory for public interfaces and important internal boundaries.
6. Background jobs, consumers, webhook handlers, and event processors MUST be idempotent where relevant.
7. Shared code in the monorepo MUST remain small, stable, and intentionally governed.
8. Shared libraries MUST NOT become a hidden coupling mechanism between services.

---

## Article XIV — Containerization and Fargate Runtime

1. Every deployable backend service MUST be containerized.
2. Each deployable service MUST produce a versioned, reproducible container image as its primary deployment artifact.
3. Services MUST be deployable independently to AWS Fargate.
4. Container images and deployment artifacts MUST be immutable once released to an environment.
5. Service runtime assumptions MUST be compatible with stateless container execution, horizontal scaling, and rolling deployment patterns.
6. Background workers, consumers, and asynchronous processors SHOULD follow the same containerized deployment model unless an explicit exception is approved.
7. Containerization is required for deployable services, but deployable service count MUST remain intentionally limited in early stages to avoid premature operational complexity.
8. Infrastructure required for service deployment and platform operation MUST be defined and managed through Infrastructure as Code. AWS CDK is the default standard unless an explicit exception is approved.

**Rationale**: The platform runtime target is AWS Fargate, but service count must be pragmatic rather than fragmented.

---

## Article XV — Reliability, Observability, and Operations

1. Every service MUST be production-operable from day one.
2. Every service MUST provide:
   - health checks
   - readiness checks
   - structured logging
   - metrics
   - trace correlation
3. Every async workflow MUST define retry policy, timeout behavior, and dead-letter visibility.
4. Services MUST degrade gracefully when dependencies fail.
5. No feature is complete without operational visibility.
6. Production incidents must be diagnosable through logs, metrics, and traces without database forensics as the primary strategy.

---

## Article XVI — Security, Privacy, and Auditability

1. Least-privilege access is mandatory across infrastructure and application layers.
2. Secrets MUST be managed through secure cloud secret management and MUST never be hardcoded.
3. Personally identifiable information MUST be minimized, classified, and handled intentionally.
4. Access to student and family data MUST be auditable.
5. Sensitive operations MUST generate audit events.
6. Encryption in transit and at rest is mandatory.
7. Internal service trust MUST be authenticated explicitly.
8. Production data MUST never be used in development or tests without approved sanitization.
9. The architecture MUST preserve future support for:
   - consent handling
   - data export
   - data deletion
   - audit review
   - policy evolution
10. Marketing, analytics, and dataset-generation capabilities MUST follow explicit access controls and data-governance rules.
11. Sensitive student data MUST NOT be exposed for campaign or dataset use without an approved business purpose and policy basis.

---

## Article XVII — Contract and Schema Evolution

1. Public APIs and event schemas MUST be version-conscious from the start.
2. Database migrations MUST be explicit, reviewable, and tracked.
3. Event schemas MUST evolve compatibly whenever feasible.
4. Deprecations MUST include migration guidance and removal timing.
5. Generated contracts do not replace human review.

---

## Article XVIII — Performance and Latency Discipline

1. Architecture decisions MUST consider latency, but latency optimization must not violate service boundaries or correctness.
2. Authentication validation should be local to each service to reduce request latency.
3. Long-running or expensive workflows MUST move to asynchronous processing.
4. Read-heavy use cases may use dedicated read models or projections.
5. Timeout budgets and retry behavior MUST be explicit for synchronous interactions.

---

## Article XIX — Documentation and Decision Traceability

1. Every service MUST maintain a short current architecture description.
2. Every feature spec MUST identify:
   - owning bounded context
   - owning service
   - API changes
   - event changes
   - storage changes
   - test strategy
3. Significant architectural choices MUST be recorded as ADRs.
4. Any exception to this constitution MUST be explicit, time-bounded, and documented.

---

## Article XX — Simplicity and Controlled Evolution

1. Prefer the simplest architecture that preserves service boundaries and future growth.
2. A new service MUST NOT be introduced until its bounded context and operational need are clear.
3. God services are prohibited.
4. Speculative abstractions are discouraged.
5. Simplicity must not be used as an excuse for violating ownership, testing, or security rules.

---

## Governance

1. This constitution overrides convenience-based implementation choices.
2. Every specification and implementation plan MUST include a constitution compliance check.
3. If work conflicts with this constitution, the conflict must be documented before implementation.
4. Amendments to this constitution require deliberate review because they affect all future work.

---

## Non-Negotiable Summary

- shared monorepo, but strict service boundaries
- Auth0 for authentication, AdmissionZen services for authorization
- local JWT validation in each service for lower latency
- explicit family model
- one family per student
- parents have full family and student access
- parent may create student profile, student later links own identity
- event-driven integration using AWS-native messaging
- Search and Analytics owns its own projection and index layer
- search, key metrics, segmentation, and derived datasets must be projection-based
- each service owns its own data
- every deployable service must be containerized and Fargate-compatible
- service count should stay intentionally limited early on
- Infrastructure as Code required, AWS CDK as default standard
- Python plus FastAPI standard
- unit tests mandatory
- integration and contract tests required
- family-level pooled billing entitlements
- secure, auditable, observable by default

**Version**: 1.1.0 | **Ratified**: 2026-03-17 | **Last Amended**: 2026-03-17
