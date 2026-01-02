# Research: PRKS Microservice Technology Stack & Architecture

**Feature**: Pembiayaan Rekening Koran Syariah (PRKS)
**Date**: 2025-12-19
**Phase**: Phase 0 - Research & Technology Selection

## Overview

This document consolidates research findings for technology stack selection and architectural patterns for the PRKS microservice. The research addresses all "NEEDS CLARIFICATION" items from the Technical Context section of the implementation plan.

---

## 1. Programming Language & Framework Selection

### Decision: Java 17 with Spring Boot 3.2

### Rationale:
1. **Industry Standard for Banking**: Java dominates core banking systems in Indonesia and globally, ensuring easier integration with existing bank infrastructure
2. **Enterprise Maturity**: Spring Boot provides comprehensive banking-grade features out of the box:
   - Spring Data JPA for transactional operations with ACID guarantees
   - Spring Security for authentication/authorization
   - Spring Cloud for microservices patterns (circuit breaker, service discovery)
   - Excellent transaction management (critical for financial operations)
3. **Performance**: JVM's mature JIT compiler provides consistent performance for the 10,000 TPS requirement
4. **Observability**: Spring Boot Actuator provides production-ready metrics, health checks, and monitoring endpoints
5. **Talent Availability**: Large pool of Java developers in banking sector reduces hiring and maintenance risks
6. **Strong Typing**: Java's static typing prevents many runtime errors critical in financial transactions
7. **Long-term Support**: Java 17 is an LTS version with support until 2029

### Alternatives Considered:

**Node.js 18+ with NestJS**:
- Pros: Fast development, good async I/O for high concurrency, TypeScript type safety
- Cons: Less mature transaction management, V8 garbage collection can cause latency spikes under load, smaller banking ecosystem
- Rejected because: Transaction integrity and predictable latency are non-negotiable for banking

**Go 1.21+**:
- Pros: Excellent performance, low memory footprint, built-in concurrency, fast compilation
- Cons: Smaller ecosystem for banking-specific libraries, less mature ORM solutions, fewer enterprise developers
- Rejected because: Higher training cost for bank teams, less integration with existing Java banking infrastructure

### Primary Dependencies:
- **Spring Boot 3.2.x**: Core framework
- **Spring Data JPA**: Database access with ACID transactions
- **Spring Security**: Authentication and authorization
- **Spring Cloud**: Microservices patterns (Circuit Breaker, Config Server)
- **Spring Kafka/RabbitMQ**: Message queue integration
- **Hibernate**: ORM for PostgreSQL
- **Resilience4j**: Circuit breaker and rate limiting
- **Micrometer**: Metrics collection
- **Swagger/SpringDoc OpenAPI**: API documentation

---

## 2. Database & Caching Strategy

### Decision: PostgreSQL 15+ (Primary) + Redis 7+ (Cache)

### Rationale:

**PostgreSQL**:
1. **ACID Compliance**: Full transactional support essential for financial operations
2. **Row-Level Locking**: Supports concurrent transaction processing without deadlocks
3. **JSON Support**: Flexible storage for audit logs and transaction metadata
4. **Proven at Scale**: Handles millions of transactions with proper indexing
5. **Point-in-Time Recovery**: Critical for financial data integrity
6. **Mature Spring Boot Integration**: Spring Data JPA provides excellent PostgreSQL support

**Redis**:
1. **Sub-millisecond Latency**: Cache facility limits and balances for real-time checks
2. **Atomic Operations**: INCR/DECR operations for balance tracking
3. **Pub/Sub**: Real-time event notifications
4. **TTL Support**: Auto-expire cached data to prevent stale reads
5. **Persistence Options**: RDB/AOF for cache recovery

### Schema Design Principles:
- Normalized schema for facilities, transactions, profit-sharing records
- Partitioning for transaction tables by date (monthly partitions)
- Indexes on customer_id, facility_number, transaction_date
- Audit tables with immutable records (append-only)
- Database-level constraints to enforce business rules

### Alternatives Considered:

**MongoDB**:
- Pros: Flexible schema, horizontal scaling
- Cons: No multi-document ACID transactions until recently, weaker consistency guarantees
- Rejected: ACID compliance is non-negotiable for financial transactions

**MySQL**:
- Pros: Widely used, good performance
- Cons: Weaker JSON support than PostgreSQL, less sophisticated query optimizer
- Rejected: PostgreSQL offers better features for complex financial queries

---

## 3. Testing Strategy & Framework

### Decision: JUnit 5 + Testcontainers + Spring Boot Test + RestAssured

### Rationale:

**JUnit 5**:
- Industry standard for Java testing
- Excellent Spring Boot integration
- Supports parameterized tests for multiple test cases
- Clear test lifecycle management

**Testcontainers**:
- Runs real PostgreSQL and Redis in Docker for integration tests
- Eliminates mocking database behavior - tests against real DB
- Ensures test environment matches production
- Critical for testing transaction isolation and ACID properties

**Spring Boot Test**:
- `@SpringBootTest` for full application context testing
- `@DataJpaTest` for repository layer tests
- `@WebMvcTest` for API controller tests
- Built-in test slicing reduces test execution time

**RestAssured**:
- Fluent API for testing REST endpoints
- JSON schema validation
- Easy to verify HTTP status codes and response structure

### Test Coverage Requirements:
- **Unit Tests**: 80%+ coverage for domain logic
- **Integration Tests**: All API endpoints, database operations, message consumers
- **Contract Tests**: Giro service integration (Pact or Spring Cloud Contract)
- **Performance Tests**: JMeter or Gatling for load testing (10,000 TPS target)

### Test Data Strategy:
- Test fixtures with realistic PRKS scenarios
- Separate test database per test class (Testcontainers isolation)
- Cleanup after each test to prevent test pollution

---

## 4. Architectural Patterns

### Decision: Hexagonal Architecture (Ports & Adapters) + Event-Driven Integration

### Rationale:

**Hexagonal Architecture**:
1. **Domain Isolation**: Business logic independent of frameworks and infrastructure
2. **Testability**: Domain can be tested without database or API layer
3. **Flexibility**: Easy to swap PostgreSQL for another database if needed
4. **Clear Boundaries**: Ports (interfaces) define contracts, Adapters implement them

**Layers**:
- **Domain Layer**: Entities (Facility, Transaction), business services, repository interfaces
- **Application Layer**: Use cases (CreateFacility, ProcessWithdrawal), orchestration
- **Infrastructure Layer**: Database repositories, message consumers, external service clients
- **API Layer**: REST controllers, DTOs, OpenAPI specification

**Event-Driven Integration**:
1. **Giro Account Events**: Listen to deposit/withdrawal events from giro service
2. **PRKS Events**: Publish events for transaction processing (for audit, reporting)
3. **Asynchronous Processing**: Decouple PRKS from giro service latency
4. **Event Sourcing for Audit**: Store all state changes as events for regulatory compliance

### Communication Patterns:

**Synchronous (REST)**:
- Facility management operations (Create, Update, Query)
- Manual transaction queries and reporting
- Health and readiness probes

**Asynchronous (Message Queue)**:
- Giro transaction events (deposit, withdrawal)
- Batch job triggers (EOD profit calculation, monthly declaration)
- Notification events (limit alerts)

### Message Queue Selection: Apache Kafka

**Rationale**:
1. **Event Log**: Persistent event log for audit trail
2. **Replay**: Can replay events for recovery or testing
3. **Scalability**: Handles high throughput (1M+ transactions/month)
4. **Ordering Guarantee**: Per-partition ordering ensures transaction sequence
5. **Spring Kafka**: Excellent Spring Boot integration

**Alternative**: RabbitMQ
- Pros: Easier to operate, good for simple pub/sub
- Cons: No built-in event log persistence, harder to replay events
- Decision: Kafka preferred for audit trail requirements

---

## 5. Transaction Processing Strategy

### Decision: Saga Pattern with Compensation + Optimistic Locking

### Rationale:

**Saga Pattern**:
- Distributed transaction between PRKS service and Giro service
- Each step is a local transaction with compensating action
- Example flow:
  1. Receive withdrawal request from giro
  2. Check PRKS limit (local transaction)
  3. Reserve PRKS funds (local transaction)
  4. Confirm to giro service (or compensate if fails)
  5. Commit PRKS withdrawal

**Optimistic Locking**:
- Use version field in Facility entity to prevent concurrent updates
- If version mismatch, retry transaction
- Prevents lost updates when multiple transactions access same facility

**Idempotency**:
- Transaction ID as idempotency key
- Duplicate transaction requests return same result without re-processing
- Critical for message queue retries

---

## 6. Profit-Sharing Calculation Algorithm

### Decision: Time-Weighted Average Balance Calculation

### Algorithm:
```
For each facility on day D:
  1. Get all transactions for day D sorted by timestamp
  2. Calculate time periods between transactions
  3. For each period:
     - time_fraction = (period_end - period_start) / total_day_seconds
     - weighted_balance += balance_in_period * time_fraction
  4. Daily profit = weighted_balance * annual_rate / 365
  5. Accumulate to monthly total
```

### Example:
```
Day: 2025-12-19
Facility: 100M IDR limit, 12% annual rate

Transactions:
09:00 - Withdrawal 50M (balance: 50M)
15:00 - Repayment 30M (balance: 20M)
18:00 - Withdrawal 10M (balance: 30M)

Calculation:
Period 1 (00:00-09:00): 0 IDR × (9/24) = 0
Period 2 (09:00-15:00): 50M × (6/24) = 12.5M
Period 3 (15:00-18:00): 20M × (3/24) = 2.5M
Period 4 (18:00-24:00): 30M × (6/24) = 7.5M

Weighted Average = 0 + 12.5M + 2.5M + 7.5M = 22.5M
Daily Profit = 22.5M × 12% / 365 = 7,397.26 IDR
```

### Implementation:
- Batch job runs at EOD (23:59)
- Uses database query to fetch day's transactions
- Stores result in `profit_sharing_records` table
- Idempotent (can re-run for same day without duplicates)

---

## 7. Security & Compliance

### Authentication:
- **JWT Tokens**: For bank officer operations (facility management)
- **API Keys**: For service-to-service calls (giro integration)
- **Token Validation**: Verify signature, expiry, audience claims

### Authorization:
- Role-based access control (RBAC)
- Roles: ADMIN (full access), OFFICER (facility management), SYSTEM (service calls)
- Endpoint-level authorization checks

### Data Protection:
- **Encryption in Transit**: TLS 1.3 for all API calls
- **Encryption at Rest**: PostgreSQL transparent data encryption (TDE)
- **PII Masking**: Customer IDs hashed in logs
- **Audit Logging**: All financial operations logged with user identity

### Regulatory Compliance:
- **Syariah Compliance**: Profit-sharing calculation auditable
- **Data Retention**: 5 years transaction history (regulatory requirement)
- **Immutable Audit Trail**: Event sourcing pattern for all state changes

---

## 8. Observability & Monitoring

### Logging:
- **Format**: JSON structured logs
- **Fields**: timestamp, level, correlation_id, service, operation, customer_id (hashed), transaction_id, duration, status
- **Tool**: ELK Stack (Elasticsearch, Logstash, Kibana) or similar

### Metrics:
- **Transaction Metrics**: throughput, latency (p50, p95, p99), error rate
- **Business Metrics**: active facilities, total outstanding, daily profit calculated
- **System Metrics**: CPU, memory, database connections, cache hit rate
- **Tool**: Prometheus + Grafana

### Tracing:
- **Distributed Tracing**: OpenTelemetry for request flow across services
- **Correlation ID**: Propagated through headers to giro service

### Alerting:
- High error rate (>1%)
- High latency (p95 > 500ms)
- Database connection pool exhausted
- Circuit breaker open

---

## 9. Deployment Strategy

### Containerization:
- **Docker**: Multi-stage build for optimized image size
- **Base Image**: Eclipse Temurin JRE 17 (official OpenJDK distribution)
- **Health Checks**: Docker health check using Spring Boot Actuator

### Orchestration:
- **Kubernetes**: Horizontal pod autoscaling based on CPU/memory
- **Deployment Strategy**: Rolling update with readiness probes
- **Configuration**: ConfigMaps for environment-specific settings
- **Secrets**: Kubernetes Secrets for database credentials, API keys

### CI/CD:
- **Build**: Maven/Gradle for dependency management and build
- **Test**: Run unit + integration tests in CI pipeline
- **Quality Gates**: SonarQube for code quality, 80%+ test coverage required
- **Container Registry**: Harbor or similar for Docker image storage

---

## 10. Performance Optimization

### Strategies:
1. **Database Indexing**: Composite indexes on (customer_id, facility_number, transaction_date)
2. **Connection Pooling**: HikariCP with optimal pool size (30-50 connections)
3. **Caching**: Redis cache for facility limits (1-minute TTL)
4. **Async Processing**: Non-critical operations (notifications) processed asynchronously
5. **Batch Operations**: Bulk insert for profit-sharing records
6. **Query Optimization**: Use EXPLAIN ANALYZE to optimize slow queries

### Load Testing:
- **Tool**: Gatling or JMeter
- **Scenarios**:
  - Normal load: 1,000 TPS sustained
  - Peak load: 10,000 TPS for 5 minutes
  - Endurance: 5,000 TPS for 1 hour
- **Success Criteria**: p95 latency < 200ms, 0% error rate

---

## Summary of Decisions

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| **Language** | Java 17 | Banking industry standard, strong typing, mature ecosystem |
| **Framework** | Spring Boot 3.2 | Comprehensive features, transaction management, microservices support |
| **Database** | PostgreSQL 15+ | ACID compliance, proven scale, excellent JSON support |
| **Cache** | Redis 7+ | Sub-millisecond latency, atomic operations, persistence |
| **Testing** | JUnit 5 + Testcontainers | Real database testing, comprehensive coverage |
| **Architecture** | Hexagonal + Event-Driven | Domain isolation, testability, async integration |
| **Message Queue** | Apache Kafka | Event log, replay capability, audit trail |
| **Security** | JWT + API Keys + TLS 1.3 | Industry standard, defense in depth |
| **Observability** | ELK + Prometheus + OpenTelemetry | Comprehensive monitoring and tracing |
| **Deployment** | Docker + Kubernetes | Cloud-native, scalable, resilient |

---

## Next Steps

All "NEEDS CLARIFICATION" items from Technical Context have been resolved. Ready to proceed to:
- **Phase 1**: Data model design, API contract generation, quickstart guide
- **Phase 2**: Task breakdown for implementation
