# LMS AMORTIZATION - MILESTONE OVERVIEW

**Version**: 1.0  
**Date**: November 3, 2025  
**Total Milestones**: 10

---

## Milestone Dependencies

```
M1: Foundation
    ↓
M2: Parameter Management
    ↓
M3: Version Management
    ↓
M4: Migration Operations
    ↓
M5: Mid-Cycle Operations ←──┐
    ↓                        │
M6: Back-dated Transactions ─┘
    ↓
M7: Audit & Compliance
    ↓
M8: Edge Cases (parallel) ←─┐
M9: Performance (parallel) ←┼─ Can run in parallel
M10: Integration (parallel) ←┘
```

---

## MILESTONE 1: FOUNDATION
**Goal**: Establish core loan operations with single product version

**Dependencies**: None

**Scenarios**:
- S1.1: New Loan Creation
- S1.2: Regular Payment Processing
- S1.3: Loan Closure

**Success Criteria**:
- Can create loans under a product version
- Can process payments with correct amortization
- Parameters resolved with correct precedence
- Audit trail captures version used

---

## MILESTONE 2: PARAMETER MANAGEMENT
**Goal**: Implement bi-temporal parameter changes without product version migrations

**Dependencies**: M1

**Scenarios**:
- S2.1: Global Parameter Change - New Loans Only
- S2.2: Product Parameter Change - No Migration Required
- S2.4: Loan-Specific Parameter Change - Migration Required
- S2.5: Parameter Correction - Historical Value Update

**Success Criteria**:
- Parameters can change with effective dates
- Historical queries return correct values
- Corrections trigger proper adjustments
- Audit trail shows original + correction

---

[All other milestones as previously documented...]

See full milestone details in individual milestone files.
