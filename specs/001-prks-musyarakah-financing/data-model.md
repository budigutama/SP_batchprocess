# Data Model: PRKS Microservice

**Feature**: Pembiayaan Rekening Koran Syariah (PRKS)
**Date**: 2025-12-19
**Phase**: Phase 1 - Design

## Overview

This document defines the data model for the PRKS microservice, including entities, relationships, validation rules, and state transitions. The model is designed for PostgreSQL implementation with Spring Data JPA.

---

## Entity Relationship Diagram

```
┌─────────────────┐
│  FacilityConfig │ (System-wide configuration)
│  (Single Row)   │
└─────────────────┘

┌──────────────────┐         ┌──────────────────┐         ┌─────────────────────┐
│   PRKSFacility   │────────<│ PRKSTransaction  │         │ ProfitSharingRecord │
│                  │ 1     * │                  │         │                     │
│ - facilityNumber │         │ - transactionId  │         │ - recordId          │
│ - customerId     │         │ - facilityNumber │         │ - facilityNumber    │
│ - giroAccount    │         │ - type           │         │ - calculationDate   │
│ - limit          │         │ - amount         │         │ - dailyProfit       │
│ - rate           │         │ - timestamp      │         └─────────────────────┘
│ - outstanding    │         └──────────────────┘                    │
│ - status         │                                                  │
└──────────────────┘         ┌─────────────────────┐                │
        │                    │ MonthlyDeclaration  │◄───────────────┘
        │ 1                  │                     │ 1            *
        │                    │ - declarationId     │
        └───────────────────>│ - facilityNumber    │
                       1   * │ - month/year        │
                             │ - totalAmount       │
                             │ - status            │
                             └─────────────────────┘

┌──────────────────┐
│   AuditLog       │ (Cross-cutting concern)
│                  │
│ - auditId        │
│ - entityType     │
│ - entityId       │
│ - operation      │
│ - changes        │
│ - userId         │
│ - timestamp      │
└──────────────────┘
```

---

## Core Entities

### 1. PRKSFacility

Represents a PRKS financing facility for a customer.

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| facilityNumber | String(20) | PK, NOT NULL, UNIQUE | Unique facility identifier (format: PRKS-YYYYMMDD-NNNN) |
| customerId | String(50) | NOT NULL, INDEX | Customer identifier from core banking system |
| giroAccountNumber | String(20) | NOT NULL, UNIQUE | Linked giro account (one-to-one relationship) |
| facilityLimit | Decimal(18,2) | NOT NULL, > 0 | Maximum credit limit in IDR |
| profitSharingRate | Decimal(5,2) | NOT NULL, 0-100 | Annual profit-sharing rate (e.g., 12.50 for 12.5%) |
| startDate | Date | NOT NULL | Facility activation date |
| endDate | Date | NOT NULL | Facility expiration date |
| currentOutstanding | Decimal(18,2) | NOT NULL, >= 0 | Current outstanding balance in IDR |
| availableLimit | Decimal(18,2) | NOT NULL, >= 0 | Calculated: facilityLimit - currentOutstanding |
| status | Enum | NOT NULL | ACTIVE, EXPIRED, FROZEN, CLOSED |
| version | Long | NOT NULL | Optimistic locking version |
| createdAt | Timestamp | NOT NULL | Record creation timestamp |
| createdBy | String(50) | NOT NULL | User ID who created facility |
| updatedAt | Timestamp | NOT NULL | Last update timestamp |
| updatedBy | String(50) | NOT NULL | User ID who last updated |

**Constraints**:
- `CHECK (endDate > startDate)`
- `CHECK (facilityLimit > 0)`
- `CHECK (profitSharingRate >= 0 AND profitSharingRate <= 100)`
- `CHECK (currentOutstanding >= 0)`
- `CHECK (currentOutstanding <= facilityLimit)`
- `CHECK (availableLimit = facilityLimit - currentOutstanding)`

**Indexes**:
- `idx_facility_customer` on (customerId)
- `idx_facility_giro` on (giroAccountNumber)
- `idx_facility_status` on (status)
- `idx_facility_enddate` on (endDate) for expiration checks

**State Transitions**:
```
[ACTIVE] ──> [FROZEN] (manual freeze by admin)
[ACTIVE] ──> [EXPIRED] (endDate reached)
[ACTIVE] ──> [CLOSED] (manual closure after full settlement)
[FROZEN] ──> [ACTIVE] (manual unfreeze)
[FROZEN] ──> [CLOSED] (manual closure)
[EXPIRED] ──> [CLOSED] (after full settlement)

Note: CLOSED is terminal state
```

**Business Rules**:
1. New withdrawals allowed only when status = ACTIVE
2. Repayments accepted in any status except CLOSED
3. Status auto-changes to EXPIRED when current_date > endDate (batch job)
4. Cannot delete facility, only close it (audit requirement)

---

### 2. PRKSTransaction

Records all PRKS facility transactions.

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| transactionId | String(30) | PK, NOT NULL, UNIQUE | Unique transaction ID (UUID or generated) |
| facilityNumber | String(20) | FK, NOT NULL, INDEX | Reference to PRKSFacility |
| transactionType | Enum | NOT NULL | WITHDRAWAL, REPAYMENT, PROFIT_DEDUCTION |
| amount | Decimal(18,2) | NOT NULL, > 0 | Transaction amount in IDR |
| transactionTimestamp | Timestamp | NOT NULL, INDEX | When transaction occurred |
| giroAccountNumber | String(20) | NOT NULL | Linked giro account |
| giroTransactionRef | String(50) | NULLABLE, INDEX | Reference to giro transaction (for reconciliation) |
| balanceBefore | Decimal(18,2) | NOT NULL | Outstanding balance before transaction |
| balanceAfter | Decimal(18,2) | NOT NULL | Outstanding balance after transaction |
| status | Enum | NOT NULL | PENDING, COMPLETED, FAILED, REVERSED |
| failureReason | String(500) | NULLABLE | Reason if status = FAILED |
| correlationId | String(50) | NOT NULL, INDEX | Request correlation ID for tracing |
| createdAt | Timestamp | NOT NULL | Record creation timestamp |

**Constraints**:
- `CHECK (amount > 0)`
- `CHECK (balanceBefore >= 0)`
- `CHECK (balanceAfter >= 0)`
- For WITHDRAWAL: `balanceAfter = balanceBefore + amount`
- For REPAYMENT: `balanceAfter = balanceBefore - amount` (but >= 0)
- For PROFIT_DEDUCTION: `balanceAfter = balanceBefore + amount`

**Indexes**:
- `idx_transaction_facility` on (facilityNumber, transactionTimestamp DESC)
- `idx_transaction_type_timestamp` on (transactionType, transactionTimestamp)
- `idx_transaction_correlation` on (correlationId)
- `idx_transaction_giro_ref` on (giroTransactionRef)

**Partitioning Strategy**:
- Partition by month on `transactionTimestamp`
- Monthly partitions for optimal query performance
- Retention: 5 years (60 partitions)

**Idempotency**:
- Use `correlationId` + `giroTransactionRef` as idempotency key
- Duplicate transaction requests return existing record without re-processing

---

### 3. ProfitSharingRecord

Daily profit-sharing calculation records.

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| recordId | Long | PK, NOT NULL, AUTO | Auto-increment primary key |
| facilityNumber | String(20) | FK, NOT NULL, INDEX | Reference to PRKSFacility |
| calculationDate | Date | NOT NULL, INDEX | Date for which profit was calculated |
| openingOutstanding | Decimal(18,2) | NOT NULL | Outstanding at start of day |
| closingOutstanding | Decimal(18,2) | NOT NULL | Outstanding at end of day |
| weightedAvgOutstanding | Decimal(18,2) | NOT NULL | Time-weighted average for the day |
| profitSharingRate | Decimal(5,2) | NOT NULL | Rate used for calculation (snapshot) |
| dailyProfitAmount | Decimal(18,2) | NOT NULL | Calculated daily profit |
| accumulatedMonthlyTotal | Decimal(18,2) | NOT NULL | Running total for current month |
| calculationMethod | String(50) | NOT NULL | Algorithm used (e.g., "TIME_WEIGHTED_AVG") |
| calculatedAt | Timestamp | NOT NULL | When calculation was performed |
| calculatedBy | String(50) | NOT NULL | System job identifier |

**Constraints**:
- `UNIQUE (facilityNumber, calculationDate)` - One record per facility per day
- `CHECK (dailyProfitAmount >= 0)`
- `CHECK (weightedAvgOutstanding >= 0)`

**Indexes**:
- `idx_profitsharing_facility_date` on (facilityNumber, calculationDate DESC)
- `idx_profitsharing_calc_date` on (calculationDate)

**Calculation Formula**:
```
dailyProfitAmount = weightedAvgOutstanding × profitSharingRate ÷ 365
```

**Business Rules**:
1. Calculation runs daily at EOD (23:59)
2. Idempotent - can re-run for same date (upsert operation)
3. If no transactions on a day, uses opening balance as constant
4. accumulatedMonthlyTotal resets on 1st of each month

---

### 4. MonthlyDeclaration

Monthly profit-sharing settlement records.

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| declarationId | String(30) | PK, NOT NULL, UNIQUE | Unique declaration ID (format: DECL-YYYYMM-NNNN) |
| facilityNumber | String(20) | FK, NOT NULL, INDEX | Reference to PRKSFacility |
| declarationMonth | Integer | NOT NULL | Month (1-12) |
| declarationYear | Integer | NOT NULL | Year (YYYY) |
| totalProfitAmount | Decimal(18,2) | NOT NULL | Sum of daily profit for the month |
| giroBalanceAtDeclaration | Decimal(18,2) | NOT NULL | Giro balance when declaration processed |
| amountDeductedFromGiro | Decimal(18,2) | NOT NULL | Amount successfully deducted from giro |
| amountAddedToOutstanding | Decimal(18,2) | NOT NULL | Amount added to PRKS outstanding |
| deductionStatus | Enum | NOT NULL | FULLY_DEDUCTED, PARTIALLY_DEDUCTED, ADDED_TO_OUTSTANDING |
| declarationDate | Date | NOT NULL | Date when declaration was generated |
| processedAt | Timestamp | NULLABLE | When declaration was processed |
| statementReference | String(100) | NULLABLE | Reference to generated statement document |
| createdAt | Timestamp | NOT NULL | Record creation timestamp |

**Constraints**:
- `UNIQUE (facilityNumber, declarationYear, declarationMonth)`
- `CHECK (totalProfitAmount = amountDeductedFromGiro + amountAddedToOutstanding)`
- `CHECK (declarationMonth >= 1 AND declarationMonth <= 12)`

**Indexes**:
- `idx_declaration_facility_date` on (facilityNumber, declarationYear, declarationMonth)
- `idx_declaration_date` on (declarationDate)
- `idx_declaration_status` on (deductionStatus)

**Business Rules**:
1. Generated on 1st of each month for previous month
2. Processing attempts to deduct from giro first
3. Any shortfall is added to PRKS outstanding
4. Immutable once created (append-only for audit)

---

### 5. FacilityConfig

System-wide configuration for PRKS facilities (single row table).

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| configId | Integer | PK, NOT NULL | Always 1 (single row) |
| maxFacilityLimitPerCustomer | Decimal(18,2) | NOT NULL | Maximum limit per customer |
| minProfitSharingRate | Decimal(5,2) | NOT NULL | Minimum allowed rate |
| maxProfitSharingRate | Decimal(5,2) | NOT NULL | Maximum allowed rate |
| maxFacilityTermMonths | Integer | NOT NULL | Maximum facility duration in months |
| utilizationAlertThreshold | Decimal(5,2) | NOT NULL | Threshold for alert (e.g., 80 for 80%) |
| declarationDayOfMonth | Integer | NOT NULL | Day of month to run declaration (1-28) |
| eodCalculationTime | Time | NOT NULL | Time to run daily calculation (e.g., 23:59) |
| updatedAt | Timestamp | NOT NULL | Last configuration update |
| updatedBy | String(50) | NOT NULL | Admin who updated |

**Constraints**:
- `CHECK (configId = 1)` - Enforce single row
- `CHECK (minProfitSharingRate < maxProfitSharingRate)`
- `CHECK (utilizationAlertThreshold > 0 AND utilizationAlertThreshold <= 100)`

**Business Rules**:
1. Single row table - use UPDATE only, never INSERT additional rows
2. Changes take effect immediately for new facilities
3. Existing facilities retain their original parameters

---

### 6. AuditLog

Comprehensive audit trail for all entity changes.

**Attributes**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| auditId | Long | PK, NOT NULL, AUTO | Auto-increment primary key |
| entityType | String(50) | NOT NULL, INDEX | Entity name (e.g., "PRKSFacility") |
| entityId | String(50) | NOT NULL, INDEX | ID of the entity that changed |
| operation | Enum | NOT NULL | INSERT, UPDATE, DELETE |
| fieldChanges | JSONB | NULLABLE | JSON object with before/after values |
| userId | String(50) | NOT NULL | User or system that made change |
| correlationId | String(50) | NULLABLE | Request correlation ID |
| ipAddress | String(50) | NULLABLE | Source IP address |
| timestamp | Timestamp | NOT NULL, INDEX | When change occurred |

**Indexes**:
- `idx_audit_entity` on (entityType, entityId, timestamp DESC)
- `idx_audit_user` on (userId, timestamp DESC)
- `idx_audit_timestamp` on (timestamp DESC)

**Partitioning**:
- Partition by month on `timestamp`
- Retention: 7 years (regulatory requirement)

**Sample fieldChanges JSON**:
```json
{
  "status": {
    "before": "ACTIVE",
    "after": "FROZEN"
  },
  "updatedBy": {
    "before": "SYSTEM",
    "after": "ADMIN001"
  }
}
```

---

## Validation Rules Summary

### PRKSFacility Validation:
1. Facility limit must be positive and <= FacilityConfig.maxFacilityLimitPerCustomer
2. Profit-sharing rate must be within FacilityConfig range
3. End date must be > start date and <= start date + maxFacilityTermMonths
4. Giro account must exist and not be linked to another active facility
5. Customer cannot exceed total credit limit across all facilities

### PRKSTransaction Validation:
1. Facility must be ACTIVE for withdrawals
2. Withdrawal amount must not exceed available limit
3. Repayment amount must not exceed current outstanding
4. Transaction must have valid giroTransactionRef for reconciliation
5. Idempotency check on (correlationId + giroTransactionRef)

### ProfitSharingRecord Validation:
1. Calculation date must not be in future
2. Daily profit calculation must match formula
3. Cannot have duplicate records for same facility + date

### MonthlyDeclaration Validation:
1. Declaration month/year must not be current or future month
2. Total profit must equal sum of daily profit records
3. Cannot have duplicate declarations for same facility + month

---

## Data Integrity Mechanisms

### 1. Foreign Key Constraints:
- PRKSTransaction.facilityNumber → PRKSFacility.facilityNumber
- ProfitSharingRecord.facilityNumber → PRKSFacility.facilityNumber
- MonthlyDeclaration.facilityNumber → PRKSFacility.facilityNumber

### 2. Optimistic Locking:
- PRKSFacility uses `version` field to prevent lost updates
- Concurrent transactions on same facility will retry on version mismatch

### 3. Database Triggers:
- `before_update_facility`: Auto-update updatedAt timestamp
- `after_insert_transaction`: Validate balance calculations
- `after_update_facility`: Insert AuditLog record

### 4. Application-Level Validation:
- Spring Validation annotations on DTOs
- Custom validators for business rules
- Transaction boundaries to ensure atomicity

---

## Derived/Computed Fields

### PRKSFacility.availableLimit:
```sql
availableLimit = facilityLimit - currentOutstanding
```
- Computed field, not stored (or stored and updated via trigger)
- Used for quick limit checks

### ProfitSharingRecord.accumulatedMonthlyTotal:
```sql
SELECT SUM(dailyProfitAmount)
FROM ProfitSharingRecord
WHERE facilityNumber = ?
  AND EXTRACT(YEAR FROM calculationDate) = ?
  AND EXTRACT(MONTH FROM calculationDate) = ?
```
- Can be cached in Redis for performance

---

## Sample Data

### PRKSFacility:
```
facilityNumber: PRKS-20251219-0001
customerId: CUST-12345
giroAccountNumber: 1234567890
facilityLimit: 500000000.00
profitSharingRate: 12.50
startDate: 2025-01-01
endDate: 2025-12-31
currentOutstanding: 150000000.00
availableLimit: 350000000.00
status: ACTIVE
```

### PRKSTransaction:
```
transactionId: TXN-20251219-ABC123
facilityNumber: PRKS-20251219-0001
transactionType: WITHDRAWAL
amount: 50000000.00
transactionTimestamp: 2025-12-19 10:30:00
balanceBefore: 100000000.00
balanceAfter: 150000000.00
status: COMPLETED
```

### ProfitSharingRecord:
```
facilityNumber: PRKS-20251219-0001
calculationDate: 2025-12-18
openingOutstanding: 100000000.00
closingOutstanding: 150000000.00
weightedAvgOutstanding: 125000000.00
profitSharingRate: 12.50
dailyProfitAmount: 42808.22
```

---

## Database Migration Strategy

### Initial Schema (V1):
1. Create all tables with constraints
2. Create indexes
3. Insert default FacilityConfig row
4. Set up partitioning for transactions and audit logs

### Future Migrations:
- Use Flyway or Liquibase for versioned migrations
- Never drop columns (add `deprecated` flag instead for audit)
- Add new columns as NULLABLE first, then populate, then make NOT NULL

---

## Performance Considerations

### 1. Indexing Strategy:
- Composite indexes on frequent query patterns
- Covering indexes for read-heavy queries
- Partial indexes on status fields

### 2. Partitioning:
- Monthly partitions for transactions (improves query performance)
- Automatic partition creation via cron job
- Old partition archival after retention period

### 3. Caching:
- Redis cache for active facility limits (1-minute TTL)
- Cache invalidation on transaction commit
- Cache-aside pattern

### 4. Query Optimization:
- Use EXPLAIN ANALYZE for slow queries
- Avoid N+1 queries (use JOIN FETCH in JPA)
- Limit result sets with pagination

---

## Next Steps

Data model complete. Ready for:
1. API contract generation (OpenAPI spec)
2. JPA entity class generation
3. Database migration scripts
4. Repository interface definitions
