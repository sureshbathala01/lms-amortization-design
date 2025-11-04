# Data Model Updates for Python Architecture

**Version**: 1.0  
**Date**: November 4, 2025  
**Impact**: Schema changes required for Java+Python hybrid

---

## Summary of Changes

| Change | Type | Impact |
|--------|------|--------|
| Add `python_version` column to `loans` | Addition | Low |
| Remove `amortization_schedule` table | Deletion | High |

---

## Change 1: Add python_version Column

### Rationale

Track which Python module version was used for each loan, enabling:
- Correct version loading for back-dated transactions
- Audit trail
- Version migration tracking

### SQL Migration Script

```sql
-- Add python_version column
ALTER TABLE loans 
ADD COLUMN python_version VARCHAR(20) NOT NULL DEFAULT '1.0';

-- Add index for version queries
CREATE INDEX idx_loans_python_version 
ON loans(product_code, python_version);

-- Update existing loans (one-time migration)
UPDATE loans 
SET python_version = product_version 
WHERE python_version = '1.0';
```

### Column Definition

```sql
python_version VARCHAR(20) NOT NULL
-- Stores version like "1.2", "1.3", "2.0"
-- Same format as product_version
-- NOT NULL to ensure every loan has version
```

### Usage Example

```sql
-- Find all loans on specific Python version
SELECT loan_id, customer_id, outstanding_balance
FROM loans
WHERE product_code = 'HOME_LOAN'
  AND python_version = '1.2';

-- Count loans by Python version
SELECT product_code, python_version, COUNT(*) as loan_count
FROM loans
WHERE status = 'ACTIVE'
GROUP BY product_code, python_version;
```

---

## Change 2: Remove amortization_schedule Table

### Rationale

**Why Remove?**
1. ✅ Schedule generated on-demand using Python logic
2. ✅ No stale data (always uses current version)
3. ✅ Simpler data model
4. ✅ Easier to maintain
5. ✅ Performance acceptable (fast calculation)

**Previous Approach** (v1.0):
- Schedule generated at disbursement
- Stored as JSON in database
- Retrieved when needed
- Problem: Stale if logic changes

**New Approach** (v2.0):
- Schedule generated on-demand
- Call Python: `generate_schedule()`
- Returns schedule (not stored)
- Always correct for loan's version

### SQL Migration Script

```sql
-- CAUTION: This deletes data
-- Backup table first if needed

-- Step 1: Backup (optional)
CREATE TABLE amortization_schedule_backup AS 
SELECT * FROM amortization_schedule;

-- Step 2: Drop table
DROP TABLE IF EXISTS amortization_schedule;

-- Step 3: Drop related indexes (if any)
-- (Automatically dropped with table)
```

### Impact Assessment

**Data Loss**: YES
- All stored schedules deleted
- Cannot recover after migration

**Mitigation**:
- Schedules can be regenerated on-demand
- No permanent data loss (recalculable)
- Backup table available if needed

**Applications Affected**:
- Any queries reading from `amortization_schedule`
- Reports using schedule data
- APIs returning schedule

**Required Code Changes**:
```java
// OLD CODE (v1.0)
Schedule schedule = scheduleRepository.findByLoanId(loanId);

// NEW CODE (v2.0)
Schedule schedule = pythonClient.generateSchedule(loanId);
```

---

## Complete Schema (After Changes)

### loans Table

```sql
CREATE TABLE loans (
    -- Primary Key
    loan_id VARCHAR(50) PRIMARY KEY,
    
    -- Customer & Product
    customer_id VARCHAR(50) NOT NULL,
    product_code VARCHAR(50) NOT NULL,
    product_version VARCHAR(20) NOT NULL,
    python_version VARCHAR(20) NOT NULL,  -- NEW
    product_version_effective_date DATE NOT NULL,
    
    -- Loan Details
    principal_amount DECIMAL(15,2) NOT NULL,
    tenure INT NOT NULL,
    tenure_unit VARCHAR(10) NOT NULL,
    
    -- Dates
    disbursement_date DATE,
    creation_date TIMESTAMP NOT NULL,
    activated_date TIMESTAMP,
    
    -- Status
    status VARCHAR(20) NOT NULL,
    
    -- Audit
    created_by VARCHAR(50),
    created_by_system VARCHAR(50),
    
    -- Foreign Keys
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (product_code) REFERENCES products(product_code),
    
    -- Indexes
    INDEX idx_loans_status (status),
    INDEX idx_loans_customer (customer_id),
    INDEX idx_loans_product (product_code, product_version),
    INDEX idx_loans_python_version (product_code, python_version)  -- NEW
);
```

### loan_parameters Table (Unchanged)

```sql
-- No changes to this table
CREATE TABLE loan_parameters (
    loan_parameter_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    loan_id VARCHAR(50) NOT NULL,
    parameter_name VARCHAR(100) NOT NULL,
    parameter_value VARCHAR(500) NOT NULL,
    parameter_type VARCHAR(50) NOT NULL,
    effective_from DATE NOT NULL,
    effective_to DATE,
    transaction_time TIMESTAMP NOT NULL,
    valid_from DATE NOT NULL,
    created_by VARCHAR(50),
    
    FOREIGN KEY (loan_id) REFERENCES loans(loan_id),
    
    INDEX idx_loan_params_loan (loan_id),
    INDEX idx_loan_params_bitemporal (
        loan_id, 
        parameter_name, 
        effective_from, 
        transaction_time
    )
);
```

---

## Migration Checklist

### Pre-Migration

- [ ] Review all code using `amortization_schedule` table
- [ ] Update code to call Python `generate_schedule()` instead
- [ ] Test schedule generation via Python
- [ ] Backup `amortization_schedule` table (if needed)
- [ ] Notify stakeholders of schema changes

### Migration Execution

- [ ] Run migration during maintenance window
- [ ] Execute SQL scripts in transaction
- [ ] Verify `python_version` column added
- [ ] Verify `amortization_schedule` table dropped
- [ ] Test loan creation end-to-end
- [ ] Test schedule generation

### Post-Migration

- [ ] Monitor application logs for errors
- [ ] Verify no references to dropped table
- [ ] Performance test schedule generation
- [ ] Update documentation
- [ ] Train support team

---

## Rollback Strategy

### If Migration Fails

```sql
-- Rollback: Remove python_version column
ALTER TABLE loans DROP COLUMN python_version;

-- Rollback: Restore amortization_schedule table
CREATE TABLE amortization_schedule AS
SELECT * FROM amortization_schedule_backup;

-- Restore original state
```

### If Issues Found Post-Migration

**Option 1**: Patch code to handle missing table gracefully

**Option 2**: Restore table from backup (if critical)

**Option 3**: Generate schedules on-the-fly (recommended)

---

## Performance Considerations

### Schedule Generation Performance

**Benchmark** (typical home loan, 240 months):
- Python calculation time: ~10-20ms
- Network overhead (Java→Python): ~5-10ms
- **Total**: ~15-30ms

**Acceptable because**:
- Schedule not generated on every request
- Only when explicitly needed
- Faster than database query for complex schedules

### Database Impact

**Positive**:
- ✅ One less table to maintain
- ✅ Simpler queries
- ✅ No stale data cleanup needed

**Negative**:
- ❌ Cannot query schedule in SQL
- ❌ Must call Python for schedule data

**Net Impact**: Positive (simpler, more flexible)

---

## Testing

### Test Cases

1. **Loan Creation**:
   - Verify `python_version` populated
   - Verify no `amortization_schedule` insert attempted

2. **Schedule Generation**:
   - Call Python API for schedule
   - Verify schedule returned correctly
   - Performance test (< 50ms)

3. **Back-dated Transactions**:
   - Verify correct Python version loaded
   - Verify schedule generated with right logic

4. **Migration Validation**:
   - All existing loans have `python_version`
   - No orphaned schedule data
   - All queries work without errors

### SQL Test Queries

```sql
-- Verify python_version populated
SELECT COUNT(*) FROM loans WHERE python_version IS NULL;
-- Expected: 0

-- Verify table dropped
SELECT * FROM amortization_schedule LIMIT 1;
-- Expected: Error (table doesn't exist)

-- Verify index created
SHOW INDEX FROM loans WHERE Key_name = 'idx_loans_python_version';
-- Expected: 1 row
```

---

## Documentation Updates Required

### Update These Documents

1. **API Documentation**: Schedule endpoints now call Python
2. **Database Schema Docs**: Reflect new schema
3. **Developer Guide**: How to generate schedules
4. **Support Manual**: Troubleshooting schedule issues

---

## Related Documents

- [Architecture Comparison](../06-technical-design/architecture-comparison-java-python.md)
- [S1.1 v2.0 Flow](../04-detailed-flows/milestone-1/S1.1-new-loan-creation-v2-python.md)
- [Python Module Specification](../06-technical-design/python-module-specification.md)

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-04 | 1.0 | Initial data model updates for Python architecture |
