# Glossary

**Version**: 1.0  
**Date**: November 3, 2025

---

## Core Concepts

### Amortization
The process of spreading loan payments over time, typically with each payment consisting of principal and interest components. The system calculates how each payment is allocated.

### Bi-temporal
Data storage approach using two time dimensions:
- **Valid-time**: When a fact was true in reality
- **Transaction-time**: When the fact was recorded in the database

### Cascade Recalculation
When a back-dated transaction triggers recalculation of all subsequent transactions and amortization schedules affected by the change.

### Contract Version
The product version that was active when the loan was originated. Determines the terms and calculation logic for the loan.

### Immutability
Once a transaction is posted, it cannot be edited or deleted. Corrections are made via adjustment entries that reference the original transaction.

### Migration
The automated process of moving loans from one product version to another, including validation and re-amortization.

### Parameter
A configurable value used in loan calculations (e.g., interest rate, late fee amount). Parameters can be global, product-specific, or loan-specific.

### Parked Loan
A loan that failed migration validation and has been set aside for manual intervention before it can be migrated to a new product version.

### Product
A loan product type (e.g., "Home Loan", "Personal Loan") with specific calculation logic and parameters.

### Product Version
A specific version of a product's calculation logic, implemented as a Java class. Each version is immutable and retained forever.

### Valid-time
The date/time when an event actually occurred in the real world (e.g., when a payment was made).

### Transaction-time
The date/time when an event was recorded in the system (e.g., when a back-dated payment was posted).

---

## Technical Terms

### Adjustment Entry
A financial transaction posted to correct a previous transaction, maintaining the original for audit trail purposes.

### Class Loading (Dynamic)
The Java mechanism for loading different versions of product classes at runtime based on the loan's product version.

### Effective Date
The date from which a product version or parameter change takes effect.

### Migration Batch
An automated job that processes multiple loans for version migration, typically running during off-peak hours.

### Parameter Precedence
The hierarchy for resolving parameter values: Loan-specific → Product-specific → Global

### Posting Date
The date when a transaction was recorded in the system (transaction-time).

### Value Date
The date for which a transaction is effective (valid-time), may differ from posting date for back-dated transactions.

### Validation (Migration)
Product version-specific logic that determines whether a loan can be migrated to a new version.

---

## Business Terms

### Grace Period
The period after a payment due date during which no late fees are assessed.

### Late Fee
A charge assessed when a payment is not received by the due date (plus grace period).

### Prepayment Penalty
A fee charged when a borrower pays off the loan earlier than the scheduled term.

### Principal
The amount of money borrowed, excluding interest and fees.

### Interest
The cost of borrowing money, typically calculated as a percentage of the principal.

### Amortization Schedule
A table showing each payment's allocation between principal, interest, and fees over the life of the loan.

---

## Regulatory Terms

### Audit Trail
Complete record of all transactions and changes, with timestamps and user information, maintained for regulatory inspection.

### RBI (Reserve Bank of India)
The central banking institution of India that issues regulations for lending practices.

### Master Circular
Official guidelines issued by RBI on specific aspects of banking operations.

### Reconstruction
The ability to recreate the exact state of a loan at any historical point in time for audit purposes.

---

## Acronyms

- **ADR**: Architecture Decision Record
- **API**: Application Programming Interface
- **INR**: Indian Rupees
- **LMS**: Loan Management System
- **RBI**: Reserve Bank of India
- **SDLC**: Software Development Life Cycle
- **SLA**: Service Level Agreement
- **TBD**: To Be Determined

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-03 | 1.0 | Initial glossary |
