# PRKS Microservice - Quick Start Guide

**Feature**: Pembiayaan Rekening Koran Syariah (PRKS)
**Version**: 1.0.0
**Last Updated**: 2025-12-19

## Overview

This guide helps developers quickly understand and start working with the PRKS microservice. It covers architecture, key concepts, API usage, and development workflow.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Key Concepts](#key-concepts)
3. [Architecture](#architecture)
4. [Getting Started](#getting-started)
5. [API Quick Reference](#api-quick-reference)
6. [Common Use Cases](#common-use-cases)
7. [Development Workflow](#development-workflow)
8. [Testing Strategy](#testing-strategy)
9. [Troubleshooting](#troubleshooting)

---

## System Overview

### What is PRKS?

PRKS (Pembiayaan Rekening Koran Syariah) is an Islamic working capital financing facility based on the Musyarakah contract. It provides:

- **Seamless Liquidity**: Automatic withdrawal from facility when giro balance insufficient
- **Automatic Repayment**: Deposits to giro automatically reduce PRKS outstanding
- **Fair Profit-Sharing**: Calculated based on actual daily fund usage
- **Revolving Credit**: Borrow and repay flexibly within facility limit

### Business Value

- **For Customers**: Uninterrupted business operations without manual financing requests
- **For Bank**: Competitive Islamic banking product with automated operations
- **For Operations**: Reduced manual processing and operational risk

---

## Key Concepts

### 1. PRKS Facility

A pre-approved credit line linked to a customer's giro account with:
- **Facility Limit**: Maximum amount customer can borrow (e.g., 500M IDR)
- **Profit-Sharing Rate**: Annual rate (e.g., 12.5%) for profit calculation
- **Term**: Start and end dates for facility validity
- **Status**: ACTIVE, EXPIRED, FROZEN, or CLOSED

### 2. Automatic Integration

**Withdrawal Flow**:
```
Customer Payment > Giro Balance Check > Insufficient?
   → Withdraw from PRKS → Update Outstanding → Complete Payment
```

**Repayment Flow**:
```
Customer Deposit > Credit to Giro > Has PRKS Outstanding?
   → Reduce Outstanding → Update Available Limit
```

### 3. Profit-Sharing Calculation

**Daily Calculation** (runs at EOD 23:59):
```
Weighted Average Outstanding = Σ(Balance × Time Period) / Total Day
Daily Profit = Weighted Average × Annual Rate ÷ 365
```

**Monthly Declaration** (runs on 1st of month):
```
Total Monthly Profit = Σ(Daily Profit for Month)
Deduct from Giro → If Insufficient → Add to Outstanding
```

### 4. Transaction Types

- **WITHDRAWAL**: Customer payment exceeds giro balance, system withdraws from PRKS
- **REPAYMENT**: Customer deposit to giro reduces PRKS outstanding
- **PROFIT_DEDUCTION**: Monthly profit-sharing settled from giro or added to outstanding

---

## Architecture

### High-Level Architecture

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Giro Service   │◄───────►│  PRKS Service   │◄───────►│  PostgreSQL     │
│  (External)     │  Events │  (This Service) │  JDBC   │  (Primary DB)   │
└─────────────────┘         └─────────────────┘         └─────────────────┘
                                     │
                                     │ Cache
                                     ▼
                            ┌─────────────────┐
                            │  Redis Cache    │
                            │  (Limits/State) │
                            └─────────────────┘
                                     │
                                     │ Messages
                                     ▼
                            ┌─────────────────┐
                            │  Apache Kafka   │
                            │  (Event Stream) │
                            └─────────────────┘
```

### Service Boundaries

**PRKS Service Owns**:
- Facility management (create, update, query)
- Transaction processing (withdrawal, repayment)
- Profit-sharing calculations (daily, monthly)
- Business rules enforcement

**Giro Service Owns**:
- Giro account balance management
- Transaction authorization
- Customer authentication

**Integration Pattern**:
- Synchronous: REST APIs for queries and commands
- Asynchronous: Kafka events for transaction processing

### Layered Architecture

```
┌──────────────────────────────────────────┐
│  API Layer (REST Controllers)           │
│  - Facility endpoints                    │
│  - Transaction endpoints                 │
│  - Profit-sharing endpoints              │
└──────────────────────────────────────────┘
                  │
┌──────────────────────────────────────────┐
│  Application Layer (Use Cases)           │
│  - CreateFacility                        │
│  - ProcessWithdrawal                     │
│  - CalculateDailyProfit                  │
└──────────────────────────────────────────┘
                  │
┌──────────────────────────────────────────┐
│  Domain Layer (Business Logic)           │
│  - Entities (Facility, Transaction)      │
│  - Services (FacilityService, etc.)      │
│  - Repository Interfaces                 │
└──────────────────────────────────────────┘
                  │
┌──────────────────────────────────────────┐
│  Infrastructure Layer                    │
│  - PostgreSQL Repositories               │
│  - Redis Cache                           │
│  - Kafka Consumers/Producers             │
│  - Giro Service Client                   │
└──────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites

- **Java 17+** (OpenJDK recommended)
- **Maven 3.8+** or **Gradle 8+**
- **Docker** (for local PostgreSQL, Redis, Kafka)
- **IntelliJ IDEA** or **VS Code** with Java extensions

### Local Development Setup

#### 1. Clone Repository

```bash
git clone <repository-url>
cd prks-service
```

#### 2. Start Dependencies (Docker Compose)

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: prks_db
      POSTGRES_USER: prks_user
      POSTGRES_PASSWORD: prks_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

volumes:
  postgres_data:
```

```bash
docker-compose up -d
```

#### 3. Configure Application

```yaml
# src/main/resources/application-local.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/prks_db
    username: prks_user
    password: prks_pass
  redis:
    host: localhost
    port: 6379
  kafka:
    bootstrap-servers: localhost:9092

prks:
  giro-service:
    base-url: http://localhost:8081  # Mock for local dev
  batch-jobs:
    daily-calculation-cron: "0 59 23 * * ?"
    monthly-declaration-cron: "0 0 0 1 * ?"
```

#### 4. Run Database Migrations

```bash
# Using Flyway
mvn flyway:migrate

# Or Gradle
./gradlew flywayMigrate
```

#### 5. Run Application

```bash
# Maven
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Gradle
./gradlew bootRun --args='--spring.profiles.active=local'
```

#### 6. Verify Service

```bash
# Health check
curl http://localhost:8080/prks/v1/health

# Readiness check
curl http://localhost:8080/prks/v1/ready
```

---

## API Quick Reference

### Authentication

**JWT for User Operations** (Facility Management):
```bash
curl -H "Authorization: Bearer <jwt-token>" \
     https://api.example.com/prks/v1/facilities
```

**API Key for Service-to-Service**:
```bash
curl -H "X-API-Key: <api-key>" \
     https://api.example.com/prks/v1/internal/transactions/process-withdrawal
```

### Common Endpoints

#### 1. Create Facility

```bash
POST /prks/v1/facilities
Content-Type: application/json
Authorization: Bearer <jwt-token>

{
  "customerId": "CUST-12345",
  "giroAccountNumber": "1234567890",
  "facilityLimit": 500000000.00,
  "profitSharingRate": 12.50,
  "startDate": "2025-01-01",
  "endDate": "2025-12-31"
}

Response: 201 Created
{
  "facilityNumber": "PRKS-20251219-0001",
  "currentOutstanding": 0.00,
  "availableLimit": 500000000.00,
  "status": "ACTIVE",
  ...
}
```

#### 2. Get Facility Details

```bash
GET /prks/v1/facilities/PRKS-20251219-0001
Authorization: Bearer <jwt-token>

Response: 200 OK
{
  "facilityNumber": "PRKS-20251219-0001",
  "customerId": "CUST-12345",
  "facilityLimit": 500000000.00,
  "currentOutstanding": 150000000.00,
  "availableLimit": 350000000.00,
  "status": "ACTIVE"
}
```

#### 3. Process Withdrawal (Internal API)

```bash
POST /prks/v1/internal/transactions/process-withdrawal
Content-Type: application/json
X-API-Key: <api-key>

{
  "giroAccountNumber": "1234567890",
  "amount": 50000000.00,
  "giroTransactionRef": "GIRO-TXN-123",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}

Response: 200 OK
{
  "transactionId": "TXN-20251219-ABC123",
  "transactionType": "WITHDRAWAL",
  "amount": 50000000.00,
  "balanceBefore": 100000000.00,
  "balanceAfter": 150000000.00,
  "status": "COMPLETED"
}
```

#### 4. Process Repayment (Internal API)

```bash
POST /prks/v1/internal/transactions/process-repayment
Content-Type: application/json
X-API-Key: <api-key>

{
  "giroAccountNumber": "1234567890",
  "amount": 30000000.00,
  "giroTransactionRef": "GIRO-TXN-456",
  "correlationId": "550e8400-e29b-41d4-a716-446655440001"
}

Response: 200 OK
{
  "transactionId": "TXN-20251219-DEF456",
  "transactionType": "REPAYMENT",
  "amount": 30000000.00,
  "balanceBefore": 150000000.00,
  "balanceAfter": 120000000.00,
  "status": "COMPLETED"
}
```

#### 5. Get Transaction History

```bash
GET /prks/v1/transactions?facilityNumber=PRKS-20251219-0001&startDate=2025-12-01&endDate=2025-12-31
Authorization: Bearer <jwt-token>

Response: 200 OK
{
  "transactions": [
    {
      "transactionId": "TXN-20251219-ABC123",
      "transactionType": "WITHDRAWAL",
      "amount": 50000000.00,
      "transactionTimestamp": "2025-12-19T10:30:00Z",
      ...
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  }
}
```

#### 6. Get Facility Utilization

```bash
GET /prks/v1/facilities/PRKS-20251219-0001/utilization
Authorization: Bearer <jwt-token>

Response: 200 OK
{
  "facilityNumber": "PRKS-20251219-0001",
  "facilityLimit": 500000000.00,
  "currentOutstanding": 400000000.00,
  "availableLimit": 100000000.00,
  "utilizationPercentage": 80.00,
  "status": "ACTIVE"
}
```

---

## Common Use Cases

### Use Case 1: Customer Pays Supplier (Insufficient Giro Balance)

**Scenario**: Customer has 10M in giro, needs to pay 30M to supplier.

**Flow**:
1. Giro service receives payment request for 30M
2. Checks giro balance: 10M (insufficient)
3. Calls PRKS API: `POST /internal/transactions/process-withdrawal` with amount 20M
4. PRKS checks facility: available limit 350M (sufficient)
5. PRKS creates withdrawal transaction: outstanding increases from 0 to 20M
6. PRKS returns success to giro service
7. Giro service completes payment: 10M from giro + 20M from PRKS

**Result**: Payment succeeds, customer unaware of automatic PRKS usage.

---

### Use Case 2: Customer Receives Payment (Has PRKS Outstanding)

**Scenario**: Customer has 50M PRKS outstanding, receives 30M deposit.

**Flow**:
1. Giro service receives deposit of 30M
2. Publishes event to Kafka: `GiroDepositEvent`
3. PRKS consumer receives event
4. Calls `POST /internal/transactions/process-repayment` with amount 30M
5. PRKS reduces outstanding from 50M to 20M
6. Available limit increases from 450M to 480M

**Result**: Outstanding auto-reduced, giro balance remains 0 (all went to PRKS).

---

### Use Case 3: End of Day Profit Calculation

**Scenario**: Daily profit-sharing calculation for facility with fluctuating balance.

**Flow**:
1. Cron job triggers at 23:59
2. For each active facility:
   - Fetch all transactions for the day
   - Calculate time-weighted average outstanding
   - Apply formula: `avg_outstanding × rate ÷ 365`
   - Store in `ProfitSharingRecord` table
   - Accumulate to monthly total
3. Update metrics and logs

**Example**:
```
Facility: PRKS-20251219-0001
Rate: 12% per annum
Transactions on 2025-12-19:
- 09:00: Withdrawal 50M (balance 0→50M)
- 15:00: Repayment 30M (balance 50M→20M)
- 18:00: Withdrawal 10M (balance 20M→30M)

Time-weighted avg:
  00:00-09:00 (9h): 0M × 9/24 = 0
  09:00-15:00 (6h): 50M × 6/24 = 12.5M
  15:00-18:00 (3h): 20M × 3/24 = 2.5M
  18:00-24:00 (6h): 30M × 6/24 = 7.5M
  Total: 22.5M

Daily profit = 22.5M × 12% ÷ 365 = 7,397.26 IDR
```

---

### Use Case 4: Monthly Declaration

**Scenario**: End of December, generate profit-sharing declaration.

**Flow**:
1. Cron job runs on January 1st
2. For each facility with outstanding in December:
   - Sum all daily profit amounts for December
   - Check giro balance at declaration time
   - Attempt to deduct profit from giro
   - If giro insufficient, add shortfall to PRKS outstanding
   - Generate declaration record and statement
3. Publish notification events

**Example**:
```
Facility: PRKS-20251219-0001
December total profit: 450,000 IDR
Giro balance on Jan 1: 1,000,000 IDR

Action:
- Deduct 450,000 from giro → balance 550,000
- PRKS outstanding unchanged
- Status: FULLY_DEDUCTED
```

---

## Development Workflow

### 1. Feature Development

```bash
# Create feature branch
git checkout -b feature/add-limit-alert

# Make changes
# Write tests (TDD approach)
# Implement feature
# Run tests locally
mvn test

# Commit and push
git commit -m "feat: add utilization alert threshold"
git push origin feature/add-limit-alert

# Create pull request
```

### 2. Code Quality Checks

```bash
# Run all tests
mvn clean test

# Run integration tests
mvn verify -P integration-tests

# Check code coverage
mvn jacoco:report
# View report: target/site/jacoco/index.html

# Run static analysis
mvn sonar:sonar
```

### 3. Database Migrations

```bash
# Create new migration
# File: src/main/resources/db/migration/V2__add_alert_threshold.sql

ALTER TABLE facility_config
ADD COLUMN alert_threshold DECIMAL(5,2) DEFAULT 80.00;

# Apply migration
mvn flyway:migrate

# Verify
mvn flyway:info
```

### 4. API Contract Changes

```bash
# Update OpenAPI spec
# specs/001-prks-musyarakah-financing/contracts/openapi.yaml

# Generate server stubs (if using codegen)
mvn openapi-generator:generate

# Update implementation
# Write tests for new endpoints
# Update integration tests
```

---

## Testing Strategy

### Unit Tests

Test business logic in isolation (domain layer).

```java
@Test
void testFacilityWithdrawal_SufficientLimit() {
    // Arrange
    Facility facility = new Facility(
        limit: 100_000_000,
        outstanding: 50_000_000
    );

    // Act
    Result result = facility.withdraw(30_000_000);

    // Assert
    assertTrue(result.isSuccess());
    assertEquals(80_000_000, facility.getCurrentOutstanding());
    assertEquals(20_000_000, facility.getAvailableLimit());
}

@Test
void testFacilityWithdrawal_InsufficientLimit() {
    Facility facility = new Facility(
        limit: 100_000_000,
        outstanding: 95_000_000
    );

    Result result = facility.withdraw(10_000_000);

    assertFalse(result.isSuccess());
    assertEquals("INSUFFICIENT_LIMIT", result.getErrorCode());
}
```

### Integration Tests

Test with real database (Testcontainers).

```java
@SpringBootTest
@Testcontainers
class FacilityRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15");

    @Autowired
    FacilityRepository repository;

    @Test
    void testSaveFacility() {
        Facility facility = createTestFacility();

        Facility saved = repository.save(facility);

        assertNotNull(saved.getFacilityNumber());
        assertEquals("CUST-12345", saved.getCustomerId());
    }
}
```

### API Tests

Test REST endpoints (RestAssured).

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class FacilityApiTest {

    @LocalServerPort
    private int port;

    @Test
    void testCreateFacility() {
        given()
            .contentType("application/json")
            .header("Authorization", "Bearer " + validJwt)
            .body(createFacilityRequest)
        .when()
            .post("/prks/v1/facilities")
        .then()
            .statusCode(201)
            .body("facilityNumber", startsWith("PRKS-"))
            .body("status", equalTo("ACTIVE"));
    }
}
```

### Contract Tests

Test integration with giro service (Spring Cloud Contract or Pact).

```java
@AutoConfigureStubRunner(
    ids = "com.bjbsyariah:giro-service:+:stubs:8081",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class GiroServiceContractTest {

    @Autowired
    GiroServiceClient giroClient;

    @Test
    void testGetGiroBalance() {
        BigDecimal balance = giroClient.getBalance("1234567890");

        assertNotNull(balance);
        assertTrue(balance.compareTo(BigDecimal.ZERO) >= 0);
    }
}
```

---

## Troubleshooting

### Common Issues

#### 1. Database Connection Failure

**Symptom**: Application won't start, error "Connection refused to PostgreSQL"

**Solution**:
```bash
# Check if PostgreSQL is running
docker ps | grep postgres

# Check connection settings
# application-local.yml should match docker-compose ports

# Restart PostgreSQL
docker-compose restart postgres
```

#### 2. Transaction Failing with Optimistic Lock Exception

**Symptom**: `OptimisticLockException` when updating facility

**Cause**: Concurrent updates to same facility

**Solution**:
```java
// Retry logic already implemented in service layer
// If persistent, check for long-running transactions
// Ensure version field is included in update requests
```

#### 3. Kafka Consumer Not Processing Events

**Symptom**: Deposits to giro not reducing PRKS outstanding

**Solution**:
```bash
# Check Kafka connectivity
docker logs prks-service_kafka_1

# Verify consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 \
    --group prks-consumers --describe

# Check topic
kafka-topics --bootstrap-server localhost:9092 --list

# View messages
kafka-console-consumer --bootstrap-server localhost:9092 \
    --topic giro-deposit-events --from-beginning
```

#### 4. Slow API Response

**Symptom**: API calls taking > 1 second

**Diagnosis**:
```bash
# Check database query performance
# Enable SQL logging in application.yml
spring.jpa.show-sql: true
spring.jpa.properties.hibernate.format_sql: true

# Check for N+1 queries
# Use EXPLAIN ANALYZE in PostgreSQL

# Check Redis cache
redis-cli
> KEYS prks:facility:*
> TTL prks:facility:PRKS-20251219-0001
```

---

## Additional Resources

### Documentation
- [Data Model](./data-model.md) - Database schema and entities
- [API Reference](./contracts/openapi.yaml) - Full OpenAPI specification
- [Architecture](./plan.md) - Detailed architecture and design decisions

### Tools
- [Postman Collection](./postman/prks-api.json) - API testing collection
- [DB Migration Scripts](../prks-service/src/main/resources/db/migration/) - Flyway migrations
- [Docker Compose](../prks-service/docker-compose.yml) - Local environment setup

### Support
- **Slack**: #prks-development
- **Email**: prks-team@bjbsyariah.co.id
- **Wiki**: https://wiki.bjbsyariah.co.id/prks

---

## Next Steps

1. **Read**: [Data Model](./data-model.md) to understand entities and relationships
2. **Explore**: [OpenAPI Spec](./contracts/openapi.yaml) for full API documentation
3. **Setup**: Follow [Getting Started](#getting-started) to run locally
4. **Implement**: Pick a task from the backlog and start coding!

Happy coding! 🚀
