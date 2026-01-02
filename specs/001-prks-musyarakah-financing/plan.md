# Implementation Plan: Pembiayaan Rekening Koran Syariah (PRKS)

**Branch**: `001-prks-musyarakah-financing` | **Date**: 2025-12-19 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-prks-musyarakah-financing/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

PRKS is an Islamic working capital financing facility using Musyarakah contract that provides seamless liquidity management through automated integration between customer giro accounts and financing facilities. The system automatically withdraws from PRKS when giro balance is insufficient and automatically reduces outstanding when deposits are made. Profit-sharing is calculated based on actual daily usage with monthly declaration and settlement.

**Technical Approach**: Microservices architecture with event-driven transaction processing, RESTful APIs for facility management, and batch processing for profit-sharing calculations. Real-time integration with core banking giro account system using message queues for transaction atomicity.

## Technical Context

**Language/Version**: NEEDS CLARIFICATION (Java 17+ with Spring Boot, or Node.js 18+ with NestJS, or Go 1.21+ recommended for banking microservices)
**Primary Dependencies**: NEEDS CLARIFICATION (pending language selection - Spring Boot/Spring Cloud, NestJS/TypeORM, or Go Fiber/GORM)
**Storage**: PostgreSQL 15+ with ACID compliance for transactional integrity; Redis for caching facility limits and real-time balance tracking
**Testing**: NEEDS CLARIFICATION (JUnit 5 + Testcontainers for Java, Jest + Supertest for Node.js, or Go testing framework)
**Target Platform**: Linux containers (Docker) running on Kubernetes or similar orchestration platform; cloud-native deployment
**Project Type**: Backend microservices with RESTful APIs - no frontend in this feature scope
**Performance Goals**: 10,000 transactions per hour; <200ms p95 latency for transaction processing; <1 minute for deposit-to-repayment cycle
**Constraints**: ACID transaction guarantees; zero balance discrepancies; 99.9% availability during business hours; Syariah compliance audit trail
**Scale/Scope**: Support 10,000+ active PRKS facilities; handle 1M+ transactions per month; maintain 5 years transaction history; real-time processing with batch end-of-day calculations

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### I. Single Responsibility ✅ PASS
**Requirement**: Each microservice must own one bounded context or business capability. Services must be independently deployable and maintainable. No shared databases between services.

**Assessment**: PRKS service owns the "PRKS Financing" bounded context including facility management, transaction processing, and profit-sharing calculations. This is a distinct business capability separate from core banking services. Integration with giro accounts will be through APIs/events, not direct database access.

**Action**: Proceed - aligns with single responsibility principle.

---

### II. API Contract ✅ PASS
**Requirement**: All services expose RESTful HTTP APIs with OpenAPI/Swagger documentation. Request/response must use JSON format. Versioning required in URL path (e.g., /v1/resource).

**Assessment**: Will implement RESTful APIs with JSON request/response for all operations. API versioning will use /v1/ prefix. OpenAPI 3.0 specification will be generated in Phase 1.

**Action**: Proceed - will comply fully in Phase 1 contract generation.

---

### III. Error Handling ✅ PASS
**Requirement**: Standardized error responses with HTTP status codes (2xx, 4xx, 5xx). Error payload must include: error code, message, timestamp. No sensitive data in error messages.

**Assessment**: All APIs will return standardized error responses with appropriate HTTP codes. Error payloads will include error_code (e.g., "INSUFFICIENT_LIMIT"), human-readable message, and ISO 8601 timestamp. Financial details and internal system information will be excluded from error messages.

**Action**: Proceed - will implement in Phase 1 contract design.

---

### IV. Health & Readiness ✅ PASS
**Requirement**: All services must implement /health (liveness) and /ready (readiness) endpoints. Return 200 OK when healthy/ready, 503 otherwise. Include basic service metadata in response.

**Assessment**: Will implement standard health endpoints:
- GET /health - Liveness probe (checks if service is running)
- GET /ready - Readiness probe (checks database connectivity, Redis availability, external dependencies)
Both will return service version, uptime, and dependency status.

**Action**: Proceed - will include in API contract specification.

---

### V. Observability ✅ PASS
**Requirement**: Structured logging required (JSON format). Each request must have unique correlation ID propagated across services. Metrics for response time, error rate, and request count mandatory.

**Assessment**: Will implement:
- Structured JSON logging with correlation IDs for request tracing
- Correlation ID propagated to giro service integration calls
- Metrics instrumentation: transaction_processing_duration, api_request_count, error_rate, profit_sharing_calculation_duration
- Log entries will include: timestamp, level, correlation_id, service, operation, customer_id (hashed), transaction_id

**Action**: Proceed - observability will be built into service design.

---

### Security Requirements ✅ PASS
**Requirement**:
- API authentication required (API keys, JWT, or OAuth2)
- Authorization checks at service boundaries
- Sensitive data encrypted in transit (HTTPS/TLS)
- Input validation on all endpoints
- Protection against common vulnerabilities

**Assessment**: Will implement:
- JWT-based authentication for bank officer operations (facility setup)
- API key authentication for internal service-to-service calls
- TLS 1.3 for all API communication
- Input validation using schema validation (OpenAPI validation middleware)
- Parameterized queries to prevent SQL injection
- Rate limiting to prevent abuse
- Audit logging for all financial transactions

**Action**: Proceed - security requirements align with banking standards.

---

### Communication Standards ✅ PASS
**Requirement**:
- Synchronous: REST over HTTP/HTTPS
- Asynchronous: Message queues or event streams when eventual consistency acceptable
- Circuit breaker pattern for external service calls
- Schema validation for all API requests/responses
- Breaking changes require new API version

**Assessment**: Will implement:
- REST APIs for facility management (synchronous operations)
- Message queue (RabbitMQ/Kafka) for giro transaction events (asynchronous integration)
- Circuit breaker for giro service calls to prevent cascading failures
- Request/response schema validation via OpenAPI
- API versioning strategy with backward compatibility

**Action**: Proceed - communication patterns align with constitution.

---

### Pre-Phase 0 Gate: ✅ PASS

All constitution requirements can be satisfied. No violations requiring justification. Proceeding to Phase 0 Research.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
prks-service/                              # PRKS Microservice root
├── src/
│   ├── api/                               # REST API controllers/handlers
│   │   ├── v1/                            # API version 1
│   │   │   ├── facilities/                # Facility management endpoints
│   │   │   ├── transactions/              # Transaction endpoints
│   │   │   └── reports/                   # Reporting endpoints
│   │   └── middleware/                    # Auth, validation, logging
│   ├── domain/                            # Business logic (domain layer)
│   │   ├── entities/                      # Core entities (Facility, Transaction, etc.)
│   │   ├── repositories/                  # Repository interfaces
│   │   ├── services/                      # Business services
│   │   │   ├── facility-service/          # Facility management logic
│   │   │   ├── transaction-service/       # Transaction processing logic
│   │   │   └── profit-sharing-service/    # Profit calculation logic
│   │   └── events/                        # Domain events
│   ├── infrastructure/                    # Technical implementations
│   │   ├── database/                      # Database repositories
│   │   │   ├── postgres/                  # PostgreSQL implementation
│   │   │   └── migrations/                # Database schema migrations
│   │   ├── cache/                         # Redis cache implementation
│   │   ├── messaging/                     # Message queue integration
│   │   │   └── consumers/                 # Event consumers (giro events)
│   │   └── external/                      # External service clients
│   │       └── giro-client/               # Giro service integration
│   ├── jobs/                              # Batch processing jobs
│   │   ├── daily-profit-calculation/      # EOD profit sharing calculation
│   │   └── monthly-declaration/           # Monthly settlement
│   └── config/                            # Configuration management
│
├── tests/
│   ├── unit/                              # Unit tests
│   │   ├── domain/                        # Business logic tests
│   │   └── services/                      # Service layer tests
│   ├── integration/                       # Integration tests
│   │   ├── api/                           # API endpoint tests
│   │   ├── database/                      # Database integration tests
│   │   └── messaging/                     # Event processing tests
│   └── contract/                          # Contract tests
│       └── giro-service/                  # Giro service contract tests
│
├── contracts/                             # OpenAPI specifications
│   └── openapi.yaml                       # API contract definition
│
├── deployment/                            # Deployment configurations
│   ├── docker/                            # Docker configurations
│   │   └── Dockerfile
│   └── k8s/                               # Kubernetes manifests
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml
│
└── docs/                                  # Additional documentation
    └── architecture/                      # Architecture diagrams
```

**Structure Decision**: Selected microservice architecture (backend service only) because:
1. PRKS is a backend financial service with no UI components
2. Single microservice owning the PRKS bounded context
3. Clean architecture with clear separation: API layer → Domain layer → Infrastructure layer
4. Supports independent deployment and scaling
5. Facilitates testing at unit, integration, and contract levels
6. Technology-agnostic structure can be implemented in Java/Spring Boot, Node.js/NestJS, or Go

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

**No violations detected.** All constitution requirements are satisfied by the design.

---

## Post-Phase 1 Constitution Re-Check

### Design Artifacts Completed:
- ✅ Research document with technology decisions
- ✅ Data model with 6 core entities
- ✅ OpenAPI 3.0 specification with 15+ endpoints
- ✅ Quickstart guide with architecture and examples

### Constitution Compliance Verification:

#### I. Single Responsibility ✅ CONFIRMED
**Design Evidence**: PRKS service is clearly bounded:
- Owns 6 entities: PRKSFacility, PRKSTransaction, ProfitSharingRecord, MonthlyDeclaration, FacilityConfig, AuditLog
- No shared database with giro service (integration via Kafka events and REST APIs)
- Clean domain boundaries defined in data model

#### II. API Contract ✅ CONFIRMED
**Design Evidence**: OpenAPI specification includes:
- 15 REST endpoints with JSON request/response
- All paths versioned with /v1/ prefix
- Complete schema definitions for all DTOs
- Standard HTTP status codes (200, 201, 400, 401, 403, 404, 409, 422, 500, 503)

#### III. Error Handling ✅ CONFIRMED
**Design Evidence**: Standardized ErrorResponse schema:
```yaml
ErrorResponse:
  errorCode: string (machine-readable)
  message: string (human-readable)
  timestamp: ISO 8601 datetime
  details: object (additional context)
```
Examples provided for all error scenarios.

#### IV. Health & Readiness ✅ CONFIRMED
**Design Evidence**: Endpoints defined:
- GET /health (liveness) - returns service status, version, uptime
- GET /ready (readiness) - checks database, cache, messaging
Both return appropriate status codes (200 OK / 503 Service Unavailable)

#### V. Observability ✅ CONFIRMED
**Design Evidence**:
- Structured JSON logging specified in research.md
- Correlation ID mandatory in all transaction requests
- Metrics defined: transaction_processing_duration, api_request_count, error_rate, profit_sharing_calculation_duration
- OpenTelemetry integration for distributed tracing

#### Security Requirements ✅ CONFIRMED
**Design Evidence**:
- JWT authentication for user operations (BearerAuth in OpenAPI)
- API Key authentication for service-to-service (ApiKeyAuth in OpenAPI)
- TLS 1.3 specified for all communication
- Input validation via OpenAPI schema validation
- Audit logging entity defined in data model

#### Communication Standards ✅ CONFIRMED
**Design Evidence**:
- REST over HTTPS for all synchronous operations
- Apache Kafka for asynchronous giro events
- Circuit breaker pattern specified (Resilience4j)
- Schema validation via OpenAPI specification
- API versioning strategy (v1 in path)

### Final Gate Status: ✅ PASS

All constitution requirements are met in the design. Ready to proceed to Phase 2 (Task Generation).
