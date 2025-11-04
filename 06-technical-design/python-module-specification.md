# Python Module Specification

**Version**: 1.0  
**Date**: November 4, 2025  
**Status**: Implementation Standard

---

## Overview

Standard structure, interface, and implementation guidelines for Python product version modules.

---

## Module Structure

### File Naming Convention

```
{product_code}_v{major}_{minor}.py

Examples:
home_loan_v1_2.py
personal_loan_v2_0.py
```

### Base Class Interface

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from decimal import Decimal
from datetime import date

class ProductVersionBase(ABC):
    """
    Base class for all product version implementations
    All 14 hook methods must be implemented (can be pass/empty)
    """
    
    def __init__(self):
        self.product_code: str = None
        self.version: str = None
        self.processed_requests: Dict[str, Any] = {}
    
    # Lifecycle Event 1: Loan Creation
    def pre_loan_creation(self, request_data: Dict) -> Dict:
        """Called BEFORE loan record created"""
        pass
    
    def post_loan_creation(self, loan_data: Dict) -> Dict:
        """Called AFTER loan record committed"""
        pass
    
    # Lifecycle Event 2: Loan Closing
    def pre_loan_closing(self, loan_data: Dict) -> Dict:
        pass
    
    def post_loan_closing(self, loan_data: Dict) -> Dict:
        pass
    
    # Lifecycle Event 3: Payment Posting
    def pre_payment_posting(self, loan_data: Dict, payment_data: Dict) -> Dict:
        """Handles all payment types via metadata"""
        pass
    
    def post_payment_posting(self, loan_data: Dict, payment_data: Dict, 
                            transaction_data: Dict) -> Dict:
        pass
    
    # Lifecycle Event 4: Rescheduling
    def pre_rescheduling(self, loan_data: Dict, reschedule_data: Dict) -> Dict:
        pass
    
    def post_rescheduling(self, loan_data: Dict) -> Dict:
        pass
    
    # Lifecycle Event 5: Restructuring
    def pre_restructuring(self, old_loans: List, new_loan_data: Dict) -> Dict:
        pass
    
    def post_restructuring(self, old_loans: List, new_loan: Dict) -> Dict:
        pass
    
    # Lifecycle Event 6: Loan Parameter Change
    def pre_loan_parameter_change(self, loan_data: Dict, 
                                  parameter_change: Dict) -> Dict:
        pass
    
    def post_loan_parameter_change(self, loan_data: Dict,
                                   parameter_change: Dict) -> Dict:
        pass
    
    # Lifecycle Event 7: Derived Parameter
    def pre_derived_parameter(self, loan_data: Dict, trigger_event: str) -> Dict:
        pass
    
    def post_derived_parameter(self, loan_data: Dict, 
                              derived_parameters: Dict) -> Dict:
        pass
    
    # Utility Methods
    def generate_schedule(self, loan_data: Dict, from_date: Optional[date] = None,
                         to_date: Optional[date] = None, 
                         num_installments: Optional[int] = None) -> Dict:
        """Generate amortization schedule on-demand (not stored)"""
        pass
```

---

## Complete Implementation Example

```python
# home_loan_v1_2.py

from decimal import Decimal, ROUND_HALF_UP
from datetime import date
from dateutil.relativedelta import relativedelta
import logging

logger = logging.getLogger(__name__)

class HomeLoanV1_2(ProductVersionBase):
    """
    Home Loan Product Version 1.2
    Released: 2025-01-01
    
    Features:
    - Standard reducing balance amortization
    - Monthly EMI payments
    - Prepayment allowed with penalty
    """
    
    def __init__(self):
        super().__init__()
        self.product_code = "HOME_LOAN"
        self.version = "1.2"
        
        # Business configuration
        self.max_principal = Decimal('10000000')  # 1 crore
        self.min_principal = Decimal('500000')     # 5 lakhs
        self.max_tenure = 360  # 30 years
        self.min_tenure = 12   # 1 year
        self.max_interest_rate = Decimal('15.0')
        self.min_interest_rate = Decimal('6.0')
    
    def pre_loan_creation(self, request_data: Dict) -> Dict:
        """Validate and calculate loan creation values"""
        request_id = request_data['request_id']
        
        # Idempotency check
        if request_id in self.processed_requests:
            logger.info(f"Returning cached result for {request_id}")
            return self.processed_requests[request_id]
        
        loan_data = request_data['loan_data']
        
        try:
            # Extract parameters
            principal = Decimal(str(loan_data['principal_amount']))
            tenure = int(loan_data['tenure'])
            overrides = loan_data.get('parameter_overrides', {})
            interest_rate = Decimal(str(overrides.get('interest_rate', '8.5')))
            
            # Validations
            errors = []
            
            if principal < self.min_principal:
                errors.append({
                    'code': 'PRINCIPAL_TOO_LOW',
                    'message': f'Principal below minimum {self.min_principal}',
                    'field': 'principal_amount'
                })
            
            if principal > self.max_principal:
                errors.append({
                    'code': 'PRINCIPAL_TOO_HIGH',
                    'message': f'Principal exceeds maximum {self.max_principal}',
                    'field': 'principal_amount'
                })
            
            if tenure < self.min_tenure or tenure > self.max_tenure:
                errors.append({
                    'code': 'INVALID_TENURE',
                    'message': f'Tenure must be {self.min_tenure}-{self.max_tenure}',
                    'field': 'tenure'
                })
            
            if interest_rate < self.min_interest_rate or \
               interest_rate > self.max_interest_rate:
                errors.append({
                    'code': 'INVALID_INTEREST_RATE',
                    'message': f'Rate must be {self.min_interest_rate}-{self.max_interest_rate}%',
                    'field': 'interest_rate'
                })
            
            if errors:
                result = {
                    'request_id': request_id,
                    'status': 'error',
                    'error_type': 'BUSINESS_ERROR',
                    'valid': False,
                    'errors': errors
                }
            else:
                # Calculate EMI and totals
                monthly_rate = interest_rate / Decimal('1200')
                emi = self._calculate_emi(principal, monthly_rate, tenure)
                total_amount = emi * tenure
                total_interest = total_amount - principal
                
                result = {
                    'request_id': request_id,
                    'status': 'success',
                    'valid': True,
                    'calculated_values': {
                        'emi': float(emi.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                        'total_interest': float(total_interest.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                        'total_amount': float(total_amount.quantize(Decimal('0.01'), ROUND_HALF_UP))
                    },
                    'parameters': {
                        'effective_rate': float(interest_rate),
                        'processing_fee_percent': 0.5
                    }
                }
            
            # Cache result
            self.processed_requests[request_id] = result
            return result
            
        except Exception as e:
            logger.error(f"Error in pre_loan_creation: {e}", exc_info=True)
            return {
                'request_id': request_id,
                'status': 'error',
                'error_type': 'TECHNICAL_ERROR',
                'errors': [{'code': 'CALCULATION_FAILED', 'message': str(e)}],
                'retryable': False
            }
    
    def post_loan_creation(self, loan_data: Dict) -> Dict:
        """Post-creation actions (minimal for creation)"""
        return {
            'request_id': loan_data.get('request_id', ''),
            'status': 'success',
            'actions': []
        }
    
    def pre_payment_posting(self, loan_data: Dict, payment_data: Dict) -> Dict:
        """Calculate payment allocation"""
        payment_type = payment_data['type']
        
        if payment_type == 'DISBURSEMENT':
            return self._handle_disbursement(loan_data, payment_data)
        elif payment_type == 'RECEIVABLE':
            return self._handle_receivable(loan_data, payment_data)
        elif payment_type == 'PREPAYMENT':
            return self._handle_prepayment(loan_data, payment_data)
        else:
            return {
                'status': 'error',
                'error_type': 'BUSINESS_ERROR',
                'valid': False,
                'errors': [{'code': 'INVALID_PAYMENT_TYPE', 
                           'message': f'Unknown type: {payment_type}'}]
            }
    
    def _handle_disbursement(self, loan_data, payment_data):
        """Handle disbursement payment"""
        amount = Decimal(str(payment_data['amount']))
        principal = Decimal(str(loan_data['principal_amount']))
        
        if amount != principal:
            return {
                'status': 'error',
                'error_type': 'BUSINESS_ERROR',
                'valid': False,
                'errors': [{'code': 'AMOUNT_MISMATCH',
                           'message': 'Disbursement must equal principal'}]
            }
        
        return {
            'status': 'success',
            'valid': True,
            'payment_allocation': {
                'principal': float(amount),
                'interest': 0.0,
                'fees': 0.0
            },
            'loan_updates': {
                'outstanding_balance': float(amount),
                'status': 'ACTIVE'
            },
            'schedule_update_required': True
        }
    
    def _handle_receivable(self, loan_data, payment_data):
        """Handle customer payment (receivable)"""
        payment_amount = Decimal(str(payment_data['amount']))
        outstanding = Decimal(str(loan_data['outstanding_balance']))
        interest_rate = Decimal(str(loan_data['interest_rate']))
        
        # Calculate interest due
        monthly_rate = interest_rate / Decimal('1200')
        interest_due = outstanding * monthly_rate
        
        # Allocate payment (interest first, then principal)
        if payment_amount <= interest_due:
            interest_paid = payment_amount
            principal_paid = Decimal('0')
        else:
            interest_paid = interest_due
            principal_paid = payment_amount - interest_due
        
        new_balance = outstanding - principal_paid
        
        return {
            'status': 'success',
            'valid': True,
            'payment_allocation': {
                'principal': float(principal_paid.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                'interest': float(interest_paid.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                'fees': 0.0
            },
            'loan_updates': {
                'outstanding_balance': float(new_balance.quantize(Decimal('0.01'), ROUND_HALF_UP))
            },
            'schedule_update_required': True
        }
    
    def post_payment_posting(self, loan_data: Dict, payment_data: Dict,
                            transaction_data: Dict) -> Dict:
        """Generate updated schedule after payment"""
        schedule = self.generate_schedule(loan_data)
        return {
            'status': 'success',
            'schedule': schedule,
            'actions': []
        }
    
    def generate_schedule(self, loan_data: Dict, from_date: Optional[date] = None,
                         to_date: Optional[date] = None,
                         num_installments: Optional[int] = None) -> Dict:
        """Generate amortization schedule"""
        principal = Decimal(str(loan_data.get('outstanding_balance',
                                              loan_data['principal_amount'])))
        interest_rate = Decimal(str(loan_data['interest_rate']))
        tenure = int(loan_data['tenure'])
        start_date = date.fromisoformat(loan_data.get('disbursement_date',
                                                      loan_data['creation_date']))
        
        monthly_rate = interest_rate / Decimal('1200')
        emi = self._calculate_emi(principal, monthly_rate, tenure)
        
        installments = []
        balance = principal
        
        for i in range(1, tenure + 1):
            due_date = start_date + relativedelta(months=i)
            
            interest = balance * monthly_rate
            principal_payment = emi - interest
            balance = balance - principal_payment
            
            if balance < Decimal('0.01'):
                principal_payment = principal_payment + balance
                balance = Decimal('0')
            
            installments.append({
                'number': i,
                'due_date': due_date.isoformat(),
                'principal_due': float(principal_payment.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                'interest_due': float(interest.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                'total_due': float(emi.quantize(Decimal('0.01'), ROUND_HALF_UP)),
                'outstanding_balance': float(balance.quantize(Decimal('0.01'), ROUND_HALF_UP))
            })
        
        total_interest = sum(inst['interest_due'] for inst in installments)
        total_principal = sum(inst['principal_due'] for inst in installments)
        
        return {
            'loan_id': loan_data['loan_id'],
            'generated_date': date.today().isoformat(),
            'installments': installments,
            'summary': {
                'total_installments': tenure,
                'total_principal': total_principal,
                'total_interest': total_interest,
                'total_amount': total_principal + total_interest
            }
        }
    
    def _calculate_emi(self, principal: Decimal, monthly_rate: Decimal,
                      tenure: int) -> Decimal:
        """
        Calculate EMI using formula:
        EMI = P * r * (1 + r)^n / ((1 + r)^n - 1)
        """
        if monthly_rate == 0:
            return principal / tenure
        
        factor = (1 + monthly_rate) ** tenure
        emi = principal * monthly_rate * factor / (factor - 1)
        
        return emi.quantize(Decimal('0.01'), ROUND_HALF_UP)
    
    # Stub implementations for other lifecycle events
    def pre_loan_closing(self, loan_data: Dict) -> Dict:
        return {'status': 'success', 'valid': True}
    
    def post_loan_closing(self, loan_data: Dict) -> Dict:
        return {'status': 'success'}
    
    def pre_rescheduling(self, loan_data, reschedule_data):
        return {'status': 'success', 'valid': True}
    
    def post_rescheduling(self, loan_data):
        return {'status': 'success'}
    
    def pre_restructuring(self, old_loans, new_loan_data):
        return {'status': 'success', 'valid': True}
    
    def post_restructuring(self, old_loans, new_loan):
        return {'status': 'success'}
    
    def pre_loan_parameter_change(self, loan_data, parameter_change):
        return {'status': 'success', 'valid': True}
    
    def post_loan_parameter_change(self, loan_data, parameter_change):
        return {'status': 'success'}
    
    def pre_derived_parameter(self, loan_data, trigger_event):
        return {'status': 'success', 'valid': True}
    
    def post_derived_parameter(self, loan_data, derived_parameters):
        return {'status': 'success'}
    
    def _handle_prepayment(self, loan_data, payment_data):
        return {'status': 'success', 'valid': True}
```

---

## Best Practices

1. **Use Decimal for Money**: Always use `Decimal` for financial calculations
2. **Idempotency**: Cache results by request_id
3. **Error Handling**: Distinguish business vs technical errors
4. **Logging**: Comprehensive logging with correlation IDs
5. **Validation**: Validate all inputs thoroughly
6. **Documentation**: Document business rules in code

---

## Flask Application Template

```python
# app.py
from flask import Flask, request, jsonify
import logging
import os
from home_loan_v1_2 import HomeLoanV1_2

app = Flask(__name__)
calculator = HomeLoanV1_2()

logging.basicConfig(level=os.getenv('LOG_LEVEL', 'INFO'))
logger = logging.getLogger(__name__)

PYTHON_ENGINE_TOKEN = os.getenv('PYTHON_ENGINE_TOKEN')

def verify_token():
    auth = request.headers.get('Authorization')
    return auth == f'Bearer {PYTHON_ENGINE_TOKEN}'

@app.before_request
def authenticate():
    if request.path == '/health':
        return
    if not verify_token():
        return jsonify({'error': 'Unauthorized'}), 401

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy', 'version': '1.2'}), 200

@app.route('/loan_creation/pre', methods=['POST'])
def pre_loan_creation():
    try:
        data = request.get_json()
        result = calculator.pre_loan_creation(data)
        return jsonify(result), 200
    except Exception as e:
        logger.error(f'Error: {e}', exc_info=True)
        return jsonify({'status': 'error', 'errors': [str(e)]}), 500

@app.route('/payment_posting/pre', methods=['POST'])
def pre_payment_posting():
    try:
        data = request.get_json()
        result = calculator.pre_payment_posting(
            data['loan_data'],
            data['payment_data']
        )
        return jsonify(result), 200
    except Exception as e:
        return jsonify({'status': 'error', 'errors': [str(e)]}), 500

if __name__ == '__main__':
    port = int(os.getenv('PORT', '8001'))
    app.run(host='0.0.0.0', port=port)
```

---

## Related Documents

- [Java-Python Integration](java-python-integration.md)
- [Architecture Comparison](architecture-comparison-java-python.md)
- [Deployment Guide](../07-implementation-guides/deployment-phase1-python-containers.md)

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-04 | 1.0 | Initial Python module specification |
