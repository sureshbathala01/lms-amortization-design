# Java-Python Integration Specification

**Version**: 1.0  
**Date**: November 4, 2025  
**Status**: Implementation Ready

---

## Overview

Complete integration specification between Java orchestration layer and Python calculation engine.

**Protocol**: REST API (Phase 1) → gRPC (Phase 2)  
**Format**: JSON  
**Authentication**: Shared bearer token

---

## Communication Architecture

```
Java Application                    Python Container
┌──────────────┐                   ┌─────────────────┐
│ Orchestrator ├──HTTP POST──────>│ Flask API       │
│              │  JSON Request     │                 │
│              │<─────────────────┤ Python Module   │
│              │  JSON Response    │ (versioned)     │
└──────────────┘                   └─────────────────┘
```

---

## Endpoint Pattern

```
POST /{event_type}/{hook_type}

Examples:
/loan_creation/pre
/loan_creation/post
/payment_posting/pre
/payment_posting/post
```

---

## Request/Response Formats

### 1. Loan Creation Pre-Hook

**Request**:
```json
{
  "request_id": "uuid-12345",
  "timestamp": "2025-11-04T10:30:45Z",
  "loan_data": {
    "customer_id": "CUST123456",
    "product_code": "HOME_LOAN",
    "product_version": "1.2",
    "principal_amount": 5000000,
    "tenure": 240,
    "tenure_unit": "MONTHS",
    "parameter_overrides": {
      "interest_rate": 8.5
    }
  }
}
```

**Response (Success)**:
```json
{
  "request_id": "uuid-12345",
  "status": "success",
  "valid": true,
  "calculated_values": {
    "emi": 41822.50,
    "total_interest": 5037400.00
  },
  "parameters": {
    "effective_rate": 8.5
  }
}
```

**Response (Business Error)**:
```json
{
  "request_id": "uuid-12345",
  "status": "error",
  "error_type": "BUSINESS_ERROR",
  "valid": false,
  "errors": [{
    "code": "AMOUNT_EXCEEDS_MAXIMUM",
    "message": "Principal exceeds maximum",
    "field": "principal_amount"
  }]
}
```

**Response (Technical Error)**:
```json
{
  "request_id": "uuid-12345",
  "status": "error",
  "error_type": "TECHNICAL_ERROR",
  "retryable": true,
  "errors": [{
    "code": "CALCULATION_FAILED",
    "message": "Internal error"
  }]
}
```

### 2. Payment Posting Pre-Hook

**Request**:
```json
{
  "request_id": "uuid-12347",
  "loan_data": {
    "loan_id": "LN2025110400001",
    "outstanding_balance": 5000000,
    "interest_rate": 8.5
  },
  "payment_data": {
    "type": "DISBURSEMENT",
    "amount": 5000000,
    "payment_date": "2025-11-15"
  }
}
```

**Response**:
```json
{
  "status": "success",
  "valid": true,
  "payment_allocation": {
    "principal": 5000000,
    "interest": 0,
    "fees": 0
  },
  "loan_updates": {
    "outstanding_balance": 5000000,
    "status": "ACTIVE"
  }
}
```

### 3. Post-Hook Schedule Generation

**Response**:
```json
{
  "status": "success",
  "schedule": {
    "loan_id": "LN2025110400001",
    "installments": [
      {
        "number": 1,
        "due_date": "2025-12-15",
        "principal_due": 15000,
        "interest_due": 35000,
        "total_due": 50000,
        "outstanding_balance": 4985000
      }
    ],
    "summary": {
      "total_installments": 240,
      "total_principal": 5000000,
      "total_interest": 5037400
    }
  }
}
```

---

## Error Handling

### Error Types

**Business Errors** (Deterministic):
- Validation failures
- Business rule violations
- **Action**: FAIL immediately, NO retry

**Technical Errors** (Non-deterministic):
- Network issues
- Container unavailable
- Timeouts
- **Action**: RETRY with exponential backoff

### Retry Mechanism

**Configuration**:
- Max attempts: 3
- Initial delay: 100ms
- Multiplier: 2.0
- Max delay: 2000ms

**Example**:
```
Attempt 1: Fail → Wait 100ms
Attempt 2: Fail → Wait 200ms
Attempt 3: Fail → Throw exception
```

**Java Implementation**:
```java
public PythonResponse callWithRetry(String url, Object request) {
    int attempt = 0;
    long delay = 100; // ms
    
    while (attempt < 3) {
        try {
            PythonResponse response = httpClient.post(url, request);
            
            if (response.isBusinessError()) {
                return response; // NO RETRY
            }
            
            if (response.isTechnicalError() && response.isRetryable()) {
                attempt++;
                Thread.sleep(delay);
                delay = Math.min(delay * 2, 2000);
                continue; // RETRY
            }
            
            return response; // SUCCESS
        } catch (NetworkException e) {
            attempt++;
            if (attempt >= 3) throw new PythonEngineException(e);
            Thread.sleep(delay);
            delay = Math.min(delay * 2, 2000);
        }
    }
}
```

### Circuit Breaker

**Purpose**: Prevent cascading failures

**Configuration**:
```java
@CircuitBreaker(
    failureRateThreshold = 50,     // 50% failures
    waitDuration = 60000,           // 60 seconds
    slidingWindowSize = 10          // Last 10 requests
)
```

**States**:
- CLOSED: Normal operation
- OPEN: Fail fast (container down)
- HALF_OPEN: Testing recovery

---

## Idempotency

### Requirement
All hook methods MUST be idempotent - safe to call multiple times with same input.

### Implementation

**Request ID Tracking**:
```python
class HomeLoanV1_2:
    def __init__(self):
        self.processed_requests = {}  # Cache
    
    def pre_loan_creation(self, request_data):
        request_id = request_data['request_id']
        
        # Check if already processed
        if request_id in self.processed_requests:
            return self.processed_requests[request_id]
        
        # Process request
        result = self._calculate(request_data)
        
        # Cache result (1 hour TTL)
        self.processed_requests[request_id] = result
        
        return result
```

---

## Timeout Configuration

**Java HTTP Client**:
```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .timeout(Duration.ofSeconds(30))  // Request timeout
    .build();
```

**Recommended Timeouts**:
- Connect: 5 seconds
- Pre-hook: 30 seconds
- Post-hook: 60 seconds (schedule generation)

---

## Security

### Authentication

**Shared Bearer Token**:
```java
// Java Request
HttpRequest request = HttpRequest.newBuilder()
    .header("Authorization", "Bearer " + PYTHON_ENGINE_TOKEN)
    .header("Content-Type", "application/json")
    .build();
```

```python
# Python Verification
def verify_token():
    token = request.headers.get('Authorization')
    if not token or token != f"Bearer {PYTHON_ENGINE_TOKEN}":
        abort(401)

@app.before_request
def authenticate():
    if request.path != '/health':
        verify_token()
```

### Network Isolation

```yaml
# docker-compose.yml
networks:
  lms-internal:
    driver: bridge

# Only Java and Python containers on this network
```

---

## Logging and Observability

### Correlation ID

Every request includes correlation ID for end-to-end tracing.

**Java**:
```java
String correlationId = UUID.randomUUID().toString();
MDC.put("correlationId", correlationId);

request.header("X-Correlation-ID", correlationId);
```

**Python**:
```python
correlation_id = request.headers.get('X-Correlation-ID')
logger.info("Processing", extra={'correlation_id': correlation_id})
```

### Metrics to Track

**Java Side**:
- `python.calls.total`
- `python.calls.errors.business`
- `python.calls.errors.technical`
- `python.calls.retries`
- `python.calls.latency.p95`

**Python Side**:
- `hook.execution.time`
- `hook.errors`
- `calculation.time`

---

## Testing

### Unit Tests (Java)

```java
@Test
public void testPreHookSuccess() {
    PythonResponse mockResp = new PythonResponse();
    mockResp.setValid(true);
    
    when(pythonClient.call(any())).thenReturn(mockResp);
    
    CreateLoanResponse response = orchestrator.createLoan(request);
    assertThat(response.getStatus()).isEqualTo("SUCCESS");
}
```

### Unit Tests (Python)

```python
def test_pre_loan_creation():
    calculator = HomeLoanV1_2()
    result = calculator.pre_loan_creation({
        'loan_data': {'principal_amount': 1000000}
    })
    assert result['valid'] == True
```

### Integration Tests

```java
@SpringBootTest
@Testcontainers
public class IntegrationTest {
    @Container
    private static GenericContainer python = 
        new GenericContainer("home-loan-v1-2:latest");
    
    @Test
    public void testEndToEnd() {
        // Test with real Python container
    }
}
```

---

## Performance

**Expected Latency**:
- Pre-hook: 20-50ms
- Post-hook: 50-100ms (with schedule)
- Total added: 70-150ms

**Acceptable for lifecycle events** (not high-frequency operations)

---

## Container Mapping

**Java Configuration**:
```yaml
python:
  containers:
    HOME_LOAN:
      "1.2": "http://home-loan-v1-2:8001"
      "1.3": "http://home-loan-v1-3:8002"
    PERSONAL_LOAN:
      "2.0": "http://personal-loan-v2-0:8003"
```

**Dynamic URL Resolution**:
```java
String url = pythonConfig.getContainerUrl(
    loan.getProductCode(),
    loan.getPythonVersion()
);
```

---

## Related Documents

- [Architecture Comparison](architecture-comparison-java-python.md)
- [Python Module Specification](python-module-specification.md)
- [Deployment Guide](../07-implementation-guides/deployment-phase1-python-containers.md)
- [S1.1 v2.0 Flow](../04-detailed-flows/milestone-1/S1.1-new-loan-creation-v2-python.md)

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-04 | 1.0 | Initial integration specification |
