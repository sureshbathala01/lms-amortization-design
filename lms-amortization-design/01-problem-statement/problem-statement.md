# Problem Statement: LMS Amortization Parameterization & Versioning

**Version**: 1.0  
**Date**: November 3, 2025  
**Status**: Approved

---

## Executive Summary

Building a Loan Management System (LMS) that maintains **financial correctness**, **regulatory compliance**, and **operational flexibility** through sophisticated parameterization and versioning of amortization logic.

---

## Core Business Problem

The fundamental business problem this system solves encompasses three interconnected needs:

### 1. Financial Correctness (Money Truth)
- System must accurately track who owes what to whom at any point in time
- Late-discovered transactions (back-dated payments) must be properly accounted for
- Corrections must be transparent with clear audit trail
- **NOT about data correction** - about money owed/owing correctness
- Every rupee must be accounted for with mathematical precision

### 2. Regulatory Compliance (RBI Guidelines)
- Complete audit trail required for regulator inspection
- Must show original transactions AND correction entries clearly
- Must be able to reconstruct loan state at any historical point
- Product versions never retired (lifetime retention for audit)
- Follows RBI master circulars on lending practices
- Immutability of posted transactions for audit integrity

### 3. Operational Flexibility
- Easy to launch new loan products with different logic
- Changes handled systematically (no manual corrections)
- Support multiple active product versions simultaneously
- Staged rollout of changes across products
- Minimize operational overhead

---

## Key Problem Drivers

### Why Bi-temporal Parameters?

**Primary Driver**: Ensure correct money owed/owing

**Scenarios Requiring Bi-temporal Support**:
1. **Delayed Payment Discovery**
   - Customer paid on Jan 15, but lender recorded it on Jan 30
   - System must calculate: "What should balance have been on Jan 20?"
   - Need both valid-time (Jan 15) and transaction-time (Jan 30)

2. **Regulatory Corrections**
   - Parameter value was incorrect historically
   - Must correct while maintaining audit trail
   - Show what was used AND what should have been used

3. **Late Transaction Posting**
   - Payment happened but not actioned by lender immediately
   - Back-dated posting required
   - Must preserve both actual date and posting date

**Technical Requirements**:
- **Valid-time**: When the event actually occurred (payment date)
- **Transaction-time**: When system learned about it (posting date)
- Latest transaction-time wins when multiple values exist for same valid-time

### Why Product Versioning?

**Core Principle**: Loan product logic = Java code + parameters

**Why Parameters Alone Are Insufficient**:
- Parameters handle values (interest rate, fees, percentages)
- Java code handles logic (calculation methods, business rules, workflows)
- Structural code changes cannot be parameterized
- Logic changes require versioning

**Decision Criteria for Version Increment**:
- **Human judgment required** - not algorithmic
- Developer decides at design-time if change is "significant"
- Factors considered:
  - Changes calculation results (different amortization output)?
  - Changes business rules (different eligibility/validation)?
  - Changes regulatory compliance approach?
  - Impact on existing loans?

**Version Characteristics**:
- Each version is a frozen, immutable Java class
- Versions never retired (lifetime retention)
- Multiple versions active simultaneously
- Shared libraries duplicated per version (no independent versioning)

### Why Migration Complexity?

**Two-Step Decision Process**:

**Step 1: Is change significant?**
- Design-time decision by developer
- If YES → increment version number
- If NO → parameter-only change

**Step 2: Should existing loans upgrade?**
- Also design-time decision
- Encoded in version metadata (`impactfulChange` flag)
- If YES → automatic migration batch triggered
- If NO → new version only for new loans

**Why Not Force All Loans to Migrate?**
- **Risk management**: Staged approach reduces blast radius
- **Validation requirements**: Some loans may not meet new version criteria
- **Operational constraints**: Cannot disrupt all loans simultaneously
- **Regulatory needs**: Some loans must stay on original contract terms

---

## Key Problem Constraints

### 1. Immutability Principle

**Posted transactions can NEVER be edited**

**Rationale**:
- Regulatory requirement for audit trail
- Financial integrity - no covering tracks
- Reproducibility for compliance

**Implementation**:
- Corrections done via adjustment entries
- Original + adjustment both visible
- Clear linkage between them

### 2. Bi-temporal Complexity

**Two Time Dimensions**:
- Valid-time: When event actually occurred
- Transaction-time: When system recorded it

**Resolution Rules**:
- Query "parameter value as of date X"
- If multiple values exist for same valid-time:
  - Use value with latest transaction-time
  - Represents most current understanding of history

### 3. Human Judgment in Versioning

**Not Algorithmic**:
- Cannot automate "is this significant?" decision
- Requires business context, domain knowledge
- Different products may have different thresholds

**Design-Time Decision**:
- Made during development
- Encoded in metadata
- Triggers automatic batch processes

### 4. Unlimited Active Versions

**No Version Retirement**:
- Versions live forever
- Needed for audit reconstruction
- Can have 10, 20, 50+ versions active

**Implications**:
- Class loading must handle many versions
- Storage grows over time
- Testing complexity increases

### 5. Staged Migration Approach

**Cannot Force Global Changes**:
- Even for global parameter changes
- Must version each product separately
- Migration staged per product
- Allows controlled rollout

### 6. Rare But Critical Back-dating

**Not Daily Operation**:
- System designed to minimize need
- Controlled environment
- Exception, not rule

**Must Handle Correctly**:
- When they occur, financial correctness critical
- Cascade through potentially hundreds of transactions
- Span multiple product versions

---

## Success Criteria

The system successfully solves the problem when:

1. ✓ **Always shows correct money owed/owing**
   - Balance accurate at any point in time
   - Corrections properly applied
   - No "lost" or "phantom" money

2. ✓ **Provides complete audit trail for regulators**
   - Can reconstruct any loan at any historical date
   - Original transactions visible
   - Corrections clearly linked
   - Version history tracked

3. ✓ **Handles late-discovered transactions correctly**
   - Back-dated payments processed accurately
   - Cascade recalculations work across versions
   - Adjustments properly posted

4. ✓ **Supports multiple product versions simultaneously**
   - Loans on different versions coexist
   - Each uses correct logic for its version
   - Migration between versions works reliably

5. ✓ **Enables flexible product evolution**
   - New versions easy to create
   - Migration automated with validation
   - Parked loans have resolution path

6. ✓ **Minimizes manual corrections**
   - Systematic handling of all scenarios
   - Batch processes handle bulk changes
   - Clear error messages for edge cases

7. ✓ **Preserves historical reproducibility forever**
   - Old versions accessible
   - Parameters queryable bi-temporally
   - Audit requirements met

---

## Problem Boundaries - What We're NOT Solving

**Explicit Out of Scope**:
- Real-time transaction processing (batch-oriented for migrations)
- Algorithmic determination of "significant change" (human decision)
- Version retirement/cleanup (versions live forever)
- Cross-product migration coordination (staged independently)
- General loan origination workflow (focus is on amortization)
- Payment collection mechanisms (focus is on posted payment processing)

---

## Regulatory Context

**Governing Body**: Reserve Bank of India (RBI)

**Key Requirements**:
- Master circulars on lending practices
- Interest calculation methods
- Penal interest computation rules
- Audit trail and documentation standards
- Consumer protection guidelines

**Compliance Obligations**:
- Demonstrate correct interest calculations
- Show complete transaction history
- Prove corrections were properly handled
- Maintain records for prescribed periods
- Respond to audit queries with reconstructed states

---

## Stakeholders

**Primary Stakeholders**:
- Loan Officers (create loans, process payments)
- Operations Team (handle migrations, resolve parked loans)
- Finance Team (ensure financial correctness)
- Compliance/Audit Team (regulatory reporting)
- Product Managers (define new loan products)
- Developers (implement product logic)

**Secondary Stakeholders**:
- Customers (loan borrowers - indirectly affected)
- Regulators (RBI inspectors)
- Executive Management (risk oversight)

---

## Key Assumptions

1. **Java Technology**: Product logic implemented in Java
2. **Relational Database**: Parameters and loan data in RDBMS
3. **Batch Processing**: Migrations run as scheduled batch jobs
4. **Deployment Control**: Code deployment triggers migration scheduling
5. **Version Stability**: Once deployed, version logic never changes
6. **Single Currency**: INR only (or specify if multi-currency)
7. **Business Hours**: Batch jobs run during off-peak hours
8. **Data Retention**: Unlimited retention (no archival/purging)

---

## Risks & Challenges

**Technical Risks**:
- Dynamic class loading complexity
- Bi-temporal query performance at scale
- Cascade recalculation performance for old back-dated transactions
- Concurrency control during migrations

**Operational Risks**:
- Parked loans accumulating (migration failures)
- Back-dated transactions creating customer confusion
- Version proliferation (too many active versions)
- Migration batch failures requiring recovery

**Regulatory Risks**:
- Audit reconstruction failing for old loans
- Correction entries not meeting RBI standards
- Version changes creating compliance gaps

---

## Change Log

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2025-11-03 | 1.0 | Design Team | Initial problem statement based on requirements gathering |

---

## Related Documents

- [Assumptions & Constraints](assumptions-and-constraints.md)
- [Scenario Catalog](../02-scenarios/scenario-catalog.md)
- [Milestone Overview](../03-milestones/milestone-overview.md)

---

**Document Owner**: [To be assigned]  
**Review Cycle**: Quarterly or upon major scope changes
