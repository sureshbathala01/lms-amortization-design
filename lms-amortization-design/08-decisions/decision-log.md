# Architecture Decision Records (ADR) Log

**Project**: LMS Amortization Parameterization & Versioning  
**Last Updated**: November 3, 2025

---

## Purpose
This log tracks all significant architecture and design decisions made during the project lifecycle.

---

## Decision Records

| ID | Date | Title | Status | Impact |
|----|------|-------|--------|--------|
| 001 | 2025-11-03 | Bi-temporal Parameter Approach | ✅ Accepted | High |
| 002 | TBD | Java Class Versioning Convention | ⏳ Pending | High |
| 003 | TBD | Migration Batch Framework | ⏳ Pending | Medium |
| 004 | TBD | Audit Trail Storage Strategy | ⏳ Pending | Medium |

---

## ADR-001: Bi-temporal Parameter Approach

**Date**: 2025-11-03  
**Status**: ✅ Accepted  
**Decision Makers**: Design Team

### Context
System needs to handle late-discovered transactions and parameter corrections while maintaining complete audit trail for regulatory compliance.

### Decision
Implement bi-temporal parameter storage with:
- **Valid-time**: When the parameter value was/is effective
- **Transaction-time**: When the system recorded the value
- Query: "Parameter value as-of date X" uses valid-time
- Conflict resolution: Latest transaction-time wins

### Consequences
**Positive**:
- Supports back-dated transactions correctly
- Enables historical corrections
- Maintains complete audit trail
- Meets RBI requirements

**Negative**:
- Query complexity increases
- Storage requirements higher
- Performance considerations for historical queries

**Mitigations**:
- Indexing strategy for temporal queries
- Caching for frequently accessed parameters

---

## ADR Template (for future decisions)

```markdown
## ADR-XXX: [Decision Title]

**Date**: YYYY-MM-DD  
**Status**: [Proposed | Accepted | Rejected | Deprecated | Superseded]  
**Decision Makers**: [Names/Roles]

### Context
[What is the issue we're seeing that is motivating this decision or change?]

### Decision
[What is the change we're proposing/making?]

### Consequences
**Positive**:
- [Benefit 1]
- [Benefit 2]

**Negative**:
- [Cost/Risk 1]
- [Cost/Risk 2]

**Mitigations**:
- [How we'll address negative consequences]

### Alternatives Considered
1. [Alternative 1] - [Why rejected]
2. [Alternative 2] - [Why rejected]

### References
- [Link to related documents]
- [Link to discussion]
```

---

## Change Log

| Date | Change |
|------|--------|
| 2025-11-03 | Initial decision log created with ADR-001 |
