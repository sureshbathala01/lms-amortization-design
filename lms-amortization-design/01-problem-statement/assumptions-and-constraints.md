# Assumptions & Constraints

**Version**: 1.0  
**Date**: November 3, 2025

---

## Technical Assumptions

### Technology Stack
1. **Programming Language**: Java (version TBD)
2. **Database**: Relational database (RDBMS) - specific vendor TBD
3. **Application Architecture**: Multi-tier enterprise application
4. **Batch Processing**: Scheduled job framework (specific tool TBD)
5. **Deployment**: On-premises or cloud (strategy TBD)

### System Characteristics
1. **Dynamic Class Loading**: Java ClassLoader mechanism supported
2. **Database Features**: Support for complex queries, indexing, transactions
3. **Concurrency**: Multi-threaded batch processing supported
4. **Transaction Management**: ACID properties guaranteed

---

## Business Assumptions

### Operational Model
1. **Business Hours**: Defined operational windows for batch jobs
2. **Batch Timing**: Migrations run during off-peak hours
3. **Manual Intervention**: Operations team available for parked loan resolution
4. **Code Deployment**: Controlled deployment process with testing
5. **Version Stability**: Once deployed, version logic never changes

### Data Assumptions
1. **Currency**: Indian Rupees (INR) only (unless multi-currency specified)
2. **Loan Types**: Specific loan products within scope (TBD)
3. **Data Volume**: Estimates for loans, transactions, parameters (TBD)
4. **Data Retention**: Unlimited retention - no archival or purging
5. **Historical Data**: Migration strategy for existing loans (if applicable)

### Regulatory Assumptions
1. **Governing Body**: RBI (Reserve Bank of India) regulations apply
2. **Audit Requirements**: Standard audit trail and reconstruction requirements
3. **Retention Period**: Indefinite (versions and data never deleted)
4. **Reporting**: Standard regulatory reports (specifics TBD)

---

## Constraints

### Hard Constraints (Cannot be Changed)

#### Regulatory Constraints
1. **Immutability**: Posted transactions cannot be edited
2. **Audit Trail**: Complete history must be maintained
3. **Reproducibility**: Historical states must be reconstructible
4. **RBI Compliance**: Must follow RBI lending guidelines

#### Business Constraints
1. **Financial Correctness**: Zero tolerance for calculation errors
2. **Version Preservation**: All product versions retained forever
3. **Operational Continuity**: System must handle active loans across versions
4. **Staged Migration**: Cannot force all loans to migrate simultaneously

### Soft Constraints (Can be Negotiated)

#### Performance Constraints
1. **Batch Duration**: Migrations should complete within operational window (negotiable)
2. **Query Performance**: Historical queries should return within acceptable time (negotiable)
3. **Concurrent Users**: Support for X concurrent users (number TBD)
4. **Transaction Throughput**: Y transactions per second (number TBD)

#### Operational Constraints
1. **Parked Loan Resolution**: Target resolution time for parked loans (negotiable)
2. **Back-dated Transaction Frequency**: Expected to be rare (< 1% of transactions)
3. **Version Release Frequency**: Expected cadence for new versions (negotiable)

---

## Design Principles

### Non-Negotiable Principles
1. **Single Source of Truth**: Database is authoritative for all financial data
2. **Idempotency**: Operations can be retried safely
3. **Auditability**: Every change tracked with who/what/when/why
4. **Fail-Safe**: Errors should not corrupt system state
5. **Backward Compatibility**: New versions don't break old versions

### Guiding Principles
1. **Simplicity**: Prefer simple solutions over complex ones
2. **Explicitness**: Make version/parameter resolution explicit and traceable
3. **Consistency**: Consistent patterns across all scenarios
4. **Testability**: Design for comprehensive testing
5. **Maintainability**: Code should be understandable and maintainable

---

## Scope Boundaries

### In Scope
- Amortization calculation logic
- Parameter management (bi-temporal)
- Product versioning and migration
- Transaction posting and processing
- Back-dated transaction handling
- Audit trail and reconstruction
- Batch migration operations

### Out of Scope
- Loan origination workflow (pre-disbursement)
- Payment collection mechanisms
- Customer communication
- Document generation (beyond statements)
- Credit scoring and approval
- Collateral management
- General ledger integration (unless specified)
- Multi-currency support (unless specified)

---

## Dependencies

### External Dependencies
1. **Database System**: Availability and performance
2. **Batch Job Framework**: Reliability and scheduling
3. **Authentication/Authorization**: User access control
4. **Logging/Monitoring**: Operational visibility
5. **Deployment Infrastructure**: Development, testing, production environments

### Internal Dependencies
1. **Product Management**: Definition of loan products and versions
2. **Compliance Team**: Validation of regulatory requirements
3. **Operations Team**: Handling of parked loans and exceptions
4. **Testing Team**: Comprehensive testing of all scenarios

---

## Risk Acceptance

### Accepted Risks
1. **Version Proliferation**: Accepting unlimited versions with storage/complexity costs
2. **Batch Duration**: Large migrations may take hours (accepted with monitoring)
3. **Parked Loans**: Some loans may remain parked requiring manual intervention
4. **Back-dated Complexity**: Cascade recalculations may be computationally expensive

### Mitigation Strategies
1. **Version Proliferation**: Monitoring and reporting on active version count
2. **Batch Duration**: Progress tracking, chunking, ability to pause/resume
3. **Parked Loans**: Dashboard for operations team, SLA tracking
4. **Back-dated Complexity**: Performance optimization, async processing if needed

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-03 | 1.0 | Initial assumptions and constraints document |
