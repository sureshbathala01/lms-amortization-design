# LMS AMORTIZATION PARAMETERIZATION & VERSIONING SCENARIOS

**Version**: 1.0  
**Date**: November 3, 2025  
**Total Scenarios**: 48 core + edge cases

---

## Document Purpose
This document catalogs all scenarios that the LMS amortization system must handle. These scenarios are derived from the problem statement and will be used for detailed end-to-end flow analysis.

---

## SCENARIO CATEGORIES

### Category 1: LOAN LIFECYCLE - Basic Operations

#### S1.1: New Loan Creation
**Description**: Creating a new loan account under a product
- Loan created with current product version
- Parameters resolved with precedence (loan → product → global)
- Initial amortization schedule generated
- Contract issued referencing product version

#### S1.2: Regular Payment Processing
**Description**: Processing an on-time payment for an active loan
- Payment posted on due date
- Amortization applied using loan's current product version
- Parameters resolved as-of payment date
- Schedule updated for remaining installments

#### S1.3: Loan Closure
**Description**: Final payment and loan account closure
- All obligations settled
- Final statement generated
- Loan marked inactive (but remains auditable)
- Product version preserved for historical reference

---

### Category 2: PARAMETER CHANGES

#### S2.1: Global Parameter Change - New Loans Only
**Description**: Global parameter changes, affects only new loans
- Example: Default grace period changes from 5 to 7 days
- Existing loans continue with old value (captured at creation)
- New loans created after change use new value
- No migration triggered

#### S2.2: Product Parameter Change - No Migration Required
**Description**: Product parameter changes, migrationRequired = false
- Example: Late fee description text updated
- Bi-temporal logic handles the change
- Existing loans automatically pick up new value for future calculations
- No version change, no migration batch

#### S2.3: Product Parameter Change - Migration Required
**Description**: Product parameter changes, migrationRequired = true
- Example: Minimum payment percentage changes from 2% to 2.5%
- Triggers automatic batch migration job
- All active loans under that product selected
- Validation executed, re-amortization performed
- Product version remains same (parameter change only)

#### S2.4: Loan-Specific Parameter Change - Migration Required
**Description**: Individual loan's parameter override changes
- Example: Loan's custom interest rate changes from 5% to 5.5%
- Immediate trigger (not batch)
- Migration validation executed
- Re-amortization performed immediately
- Single loan affected

#### S2.5: Parameter Correction - Historical Value Update
**Description**: Bi-temporal correction of parameter value
- Discovery that parameter value was incorrect historically
- Parameter updated with valid-from date in the past
- Triggers cascade recalculation for affected loans
- Adjustment entries created with posting-date = correction date
- All schedules regenerated from valid-from date onwards

---

[Continue with all other categories as previously documented...]

For the complete scenario catalog, see the full document in the repository.
