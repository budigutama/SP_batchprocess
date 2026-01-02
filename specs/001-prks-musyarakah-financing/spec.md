# Feature Specification: Pembiayaan Rekening Koran Syariah (PRKS)

**Feature Branch**: `001-prks-musyarakah-financing`
**Created**: 2025-12-19
**Status**: Draft
**Input**: User description: "Pembiayaan Rekening Koran Syariah (PRKS) merupakan fasilitas pembiayaan modal kerja yang dirancang untuk memberikan fleksibilitas tinggi bagi nasabah dalam mengelola kebutuhan likuiditas usahanya. Melalui akad Musyarakah, PRKS memungkinkan nasabah menarik dana sesuai kebutuhan dan mengembalikannya kapan saja selama limit masih tersedia..."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automated Fund Withdrawal from PRKS Facility (Priority: P1)

A business customer has an active PRKS facility with a limit of 500,000,000 IDR and a current giro account balance of 10,000,000 IDR. When the customer makes a payment of 30,000,000 IDR, the system automatically withdraws 20,000,000 IDR from the PRKS facility to cover the shortfall, allowing the transaction to complete successfully without manual intervention.

**Why this priority**: This is the core value proposition of PRKS - seamless liquidity management. Without this, PRKS is just a regular financing product. This enables customers to maintain business operations without interruption.

**Independent Test**: Can be fully tested by setting up a customer with PRKS facility, initiating a payment exceeding giro balance, and verifying automatic withdrawal occurs and delivers uninterrupted transaction flow.

**Acceptance Scenarios**:

1. **Given** customer has PRKS facility with 100,000,000 IDR available limit and giro balance of 5,000,000 IDR, **When** customer initiates payment of 15,000,000 IDR, **Then** system withdraws 10,000,000 IDR from PRKS facility, deducts 5,000,000 IDR from giro, payment completes successfully, and PRKS outstanding increases by 10,000,000 IDR
2. **Given** customer has PRKS facility with 50,000,000 IDR available limit and giro balance of 0 IDR, **When** customer initiates payment of 30,000,000 IDR, **Then** system withdraws 30,000,000 IDR from PRKS facility, payment completes successfully, and PRKS outstanding increases by 30,000,000 IDR
3. **Given** customer has PRKS facility with 10,000,000 IDR available limit and giro balance of 5,000,000 IDR, **When** customer initiates payment of 20,000,000 IDR, **Then** transaction is rejected with message indicating insufficient funds, and no withdrawal from PRKS occurs

---

### User Story 2 - Automatic Outstanding Reduction on Deposits (Priority: P1)

A business customer with an active PRKS outstanding balance of 50,000,000 IDR receives a customer payment of 30,000,000 IDR deposited into their giro account. The system automatically reduces the PRKS outstanding by 30,000,000 IDR, bringing the outstanding balance to 20,000,000 IDR and freeing up 30,000,000 IDR of the credit limit for future use.

**Why this priority**: This is equally critical to the core value - automatic repayment reduces customer burden and ensures optimal credit utilization. Without this, customers would need manual intervention for every repayment, defeating the purpose of revolving credit.

**Independent Test**: Can be fully tested by creating a customer with existing PRKS outstanding, processing a deposit to giro account, and verifying automatic reduction of outstanding and delivers optimized credit availability.

**Acceptance Scenarios**:

1. **Given** customer has PRKS outstanding of 80,000,000 IDR with facility limit of 100,000,000 IDR, **When** customer deposits 30,000,000 IDR to giro account, **Then** PRKS outstanding reduces to 50,000,000 IDR automatically, available limit increases to 50,000,000 IDR, and giro balance remains at 0 IDR
2. **Given** customer has PRKS outstanding of 20,000,000 IDR, **When** customer deposits 50,000,000 IDR to giro account, **Then** PRKS outstanding reduces to 0 IDR, and giro balance increases to 30,000,000 IDR
3. **Given** customer has PRKS outstanding of 100,000,000 IDR and receives multiple deposits of 10,000,000 IDR each within the same day, **When** each deposit is processed, **Then** PRKS outstanding reduces incrementally with each deposit in the order received

---

### User Story 3 - Daily Profit-Sharing Calculation Based on Actual Usage (Priority: P2)

At the end of each business day, the system calculates profit-sharing obligations for all customers with PRKS facilities based on the actual outstanding balance used that day. A customer who used an average of 75,000,000 IDR throughout the day with a profit-sharing rate of 12% per annum has their daily profit-sharing calculated as (75,000,000 × 12% ÷ 365), which is recorded and accumulated for monthly declaration.

**Why this priority**: Fair and transparent profit-sharing is essential for Syariah compliance and customer trust. However, it can be calculated in batch after core transactional features (P1) are working. Customers can still use the facility even if calculation is delayed.

**Independent Test**: Can be fully tested by simulating a day's worth of transactions for a customer, running end-of-day batch process, and verifying profit-sharing calculation matches formula based on actual fund usage.

**Acceptance Scenarios**:

1. **Given** customer has PRKS facility with 12% profit-sharing rate and maintained constant outstanding of 50,000,000 IDR throughout the day, **When** daily profit-sharing calculation runs, **Then** daily profit-sharing of (50,000,000 × 12% ÷ 365) = 16,438.36 IDR is recorded
2. **Given** customer has PRKS facility and outstanding balance fluctuated during the day (0 IDR at start, 100,000,000 IDR from 09:00-15:00, 0 IDR after 15:00), **When** daily profit-sharing calculation runs, **Then** system calculates based on weighted average outstanding and records appropriate daily profit-sharing
3. **Given** customer has zero outstanding balance throughout entire day, **When** daily profit-sharing calculation runs, **Then** daily profit-sharing recorded is 0 IDR

---

### User Story 4 - Monthly Profit-Sharing Declaration and Deduction (Priority: P2)

At the end of each month, the system generates a profit-sharing declaration statement for all customers summarizing total profit-sharing obligations for the month. For a customer with accumulated monthly profit-sharing of 500,000 IDR, the system automatically deducts this amount from their giro account on the declaration date (or adds to PRKS outstanding if giro balance is insufficient).

**Why this priority**: Monthly settlement is important for accounting and customer awareness, but the facility can operate without monthly declaration for a period. Daily calculations (P2) must work first before monthly aggregation makes sense.

**Independent Test**: Can be fully tested by running a full month simulation with daily calculations, triggering monthly declaration process, and verifying statement generation and automatic deduction delivers proper monthly settlement.

**Acceptance Scenarios**:

1. **Given** customer has accumulated 450,000 IDR profit-sharing for the month and giro balance of 1,000,000 IDR, **When** monthly declaration runs on declaration date, **Then** system generates declaration statement, deducts 450,000 IDR from giro balance, and giro balance becomes 550,000 IDR
2. **Given** customer has accumulated 300,000 IDR profit-sharing for the month and giro balance of 100,000 IDR, **When** monthly declaration runs, **Then** system generates declaration statement, deducts 100,000 IDR from giro, and adds 200,000 IDR to PRKS outstanding
3. **Given** customer has zero profit-sharing for the month (no usage), **When** monthly declaration runs, **Then** system generates statement showing 0 IDR obligation with no deduction

---

### User Story 5 - PRKS Facility Setup and Parameter Management (Priority: P3)

A bank officer needs to set up a new PRKS facility for an approved customer. The officer enters facility parameters including customer account number, facility limit (e.g., 200,000,000 IDR), profit-sharing rate (e.g., 11.5% per annum), facility start date, end date, and linked giro account. The system validates all parameters and activates the facility for immediate use.

**Why this priority**: Facility setup is necessary but can be done manually or through simplified interface initially. The transactional features (P1-P2) are more critical for customer value. This is administrative functionality that supports the core features.

**Independent Test**: Can be fully tested by completing facility setup form with valid parameters, submitting for activation, and verifying facility becomes active and ready for transactions.

**Acceptance Scenarios**:

1. **Given** officer has customer approval details, **When** officer enters facility parameters (limit: 150,000,000 IDR, rate: 10.5%, term: 12 months, linked giro: 1234567890), **Then** system validates parameters, creates facility record, and facility status becomes "Active"
2. **Given** officer enters facility with end date before start date, **When** officer submits facility setup, **Then** system rejects with validation error "End date must be after start date"
3. **Given** officer enters profit-sharing rate of 25% (exceeds bank policy maximum of 20%), **When** officer submits facility setup, **Then** system rejects with validation error indicating rate exceeds policy limits

---

### User Story 6 - Facility Usage Monitoring and Limit Alerts (Priority: P3)

A customer has utilized 90,000,000 IDR out of their 100,000,000 IDR PRKS facility limit. When the customer attempts a transaction that would require 15,000,000 IDR from PRKS (exceeding available limit of 10,000,000 IDR), the system prevents the transaction and notifies the customer of insufficient PRKS facility limit, suggesting they deposit funds or request a limit increase.

**Why this priority**: Monitoring and alerts enhance user experience but the core prevention logic is embedded in P1 withdrawal scenarios. Enhanced notifications and reporting can be added after core functionality is proven.

**Independent Test**: Can be fully tested by setting customer near facility limit, attempting transactions exceeding available limit, and verifying appropriate notifications and prevention logic.

**Acceptance Scenarios**:

1. **Given** customer has 5,000,000 IDR available PRKS limit remaining, **When** customer attempts payment requiring 10,000,000 IDR from PRKS, **Then** transaction is prevented and customer receives notification "Insufficient PRKS facility limit. Available: 5,000,000 IDR. Required: 10,000,000 IDR"
2. **Given** customer facility utilization reaches 80% of limit, **When** daily monitoring runs, **Then** customer receives alert notification "Your PRKS facility is 80% utilized. Consider depositing funds or contact bank for limit review"
3. **Given** customer has 0 IDR available limit (fully utilized), **When** customer attempts any withdrawal, **Then** system provides clear message with options to deposit funds or request limit increase

---

### Edge Cases

- **What happens when PRKS facility expires while customer has outstanding balance?** System should prevent new withdrawals but continue accepting deposits for repayment. Customer must settle outstanding or renew facility.
- **What happens when profit-sharing rate changes mid-month?** System should apply new rate from effective date forward and calculate month-end using weighted rates for respective periods.
- **How does system handle same-day multiple deposits and withdrawals?** All transactions processed in chronological order; profit-sharing calculation uses time-weighted average outstanding.
- **What happens during system downtime when giro transaction is attempted?** Transaction should queue or fail gracefully with clear message; no partial PRKS withdrawal without completing full transaction.
- **What happens if monthly declaration date falls on a holiday?** System should process on next business day with clear indication on statement.
- **What happens when linked giro account is closed or frozen?** PRKS facility should be frozen for new withdrawals; existing outstanding must be managed through alternative collection process.
- **How does system handle concurrent transactions from same customer?** System must use proper transaction locking to prevent race conditions in balance calculations.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST link each PRKS facility to exactly one giro account for automated integration
- **FR-002**: System MUST automatically check giro balance before processing any debit transaction
- **FR-003**: System MUST automatically withdraw funds from PRKS facility when giro balance is insufficient to cover transaction amount
- **FR-004**: System MUST reject transactions when combined giro balance plus available PRKS limit cannot cover transaction amount
- **FR-005**: System MUST automatically reduce PRKS outstanding balance when deposits are made to linked giro account
- **FR-006**: System MUST prioritize PRKS outstanding reduction over increasing giro balance when processing deposits
- **FR-007**: System MUST maintain real-time tracking of PRKS facility limit, outstanding balance, and available limit
- **FR-008**: System MUST record every PRKS withdrawal and repayment transaction with timestamp, amount, and transaction reference
- **FR-009**: System MUST calculate daily profit-sharing based on actual average outstanding balance for each customer
- **FR-010**: System MUST apply profit-sharing rate as annual percentage and calculate daily amount using formula: (Outstanding × Rate ÷ 365)
- **FR-011**: System MUST accumulate daily profit-sharing calculations for monthly declaration
- **FR-012**: System MUST generate monthly profit-sharing declaration statement for each customer with active PRKS usage
- **FR-013**: System MUST automatically deduct monthly profit-sharing from giro account on declaration date
- **FR-014**: System MUST add profit-sharing amount to PRKS outstanding if giro balance is insufficient for deduction
- **FR-015**: System MUST prevent new PRKS withdrawals when facility has reached 100% utilization
- **FR-016**: System MUST prevent new PRKS withdrawals when facility has passed end date
- **FR-017**: System MUST support facility parameter configuration including: limit amount, profit-sharing rate, start date, end date, linked giro account
- **FR-018**: System MUST validate facility parameters against business rules during setup (e.g., rate limits, date validity, account existence)
- **FR-019**: System MUST maintain audit trail of all facility parameter changes including who made change and when
- **FR-020**: System MUST generate accounting journal entries for each PRKS transaction (withdrawal, repayment, profit-sharing)
- **FR-021**: System MUST support multiple PRKS facilities per customer, each linked to different giro accounts
- **FR-022**: System MUST calculate weighted average outstanding for profit-sharing when balance fluctuates during the day
- **FR-023**: System MUST handle month-end processing including declaration generation, statement distribution, and automatic deduction
- **FR-024**: System MUST provide facility utilization reporting showing current outstanding, limit, usage percentage, and available limit
- **FR-025**: System MUST enforce transaction atomicity - either both giro and PRKS transactions complete or both roll back

### Key Entities

- **PRKS Facility**: Represents a working capital financing agreement with attributes: facility number, customer ID, linked giro account number, facility limit amount, profit-sharing rate (annual %), start date, end date, current outstanding balance, available limit, facility status (Active, Expired, Frozen, Closed)
- **PRKS Transaction**: Records all facility activities with attributes: transaction ID, facility number, transaction type (Withdrawal, Repayment, Profit-Sharing Deduction), transaction date/time, amount, giro account reference, resulting outstanding balance, transaction status
- **Profit-Sharing Record**: Daily calculation records with attributes: record ID, facility number, calculation date, opening outstanding, closing outstanding, weighted average outstanding, profit-sharing rate applied, daily profit-sharing amount, accumulated monthly total
- **Monthly Declaration**: Monthly settlement records with attributes: declaration ID, facility number, declaration month/year, total profit-sharing amount, declaration date, deduction status (Deducted from Giro, Added to Outstanding, Pending), statement reference
- **Giro Account**: Existing customer current account with integration points: account number, current balance, linked PRKS facilities, transaction history interface
- **Facility Parameters**: Configuration defining business rules with attributes: maximum facility limit per customer segment, profit-sharing rate ranges (minimum/maximum), facility term limits, utilization thresholds for alerts

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Customer transactions complete without manual intervention when PRKS facility has available limit (100% automation for eligible transactions)
- **SC-002**: Time from deposit to PRKS outstanding reduction is under 1 minute (near real-time processing)
- **SC-003**: Profit-sharing calculations reflect actual daily fund usage with accuracy to 2 decimal places
- **SC-004**: Monthly declaration statements generated for all active customers within 1 business day of month-end
- **SC-005**: Zero discrepancies between PRKS outstanding balance and sum of transaction history for any facility
- **SC-006**: System processes concurrent transactions from same customer without balance inconsistencies (100% transaction integrity)
- **SC-007**: 95% of facility setup requests completed and activated within 10 minutes of submission
- **SC-008**: Customers can view real-time facility utilization status with less than 5-second delay from transaction completion
- **SC-009**: Transaction rejection messages provide clear reason and available options within 3 seconds of attempt
- **SC-010**: System handles peak load of 10,000 PRKS transactions per hour without performance degradation
- **SC-011**: All accounting journal entries generated within 5 minutes of source transaction completion
- **SC-012**: Monthly declaration deduction success rate of 90% on first attempt (remaining 10% handled via add to outstanding)
