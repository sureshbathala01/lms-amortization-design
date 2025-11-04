# Deployment Guide: Phase 1 - Python Containers

**Version**: 1.0  
**Date**: November 4, 2025  
**Target**: Docker Compose (Simple Start)

---

## Overview

Phase 1 deployment uses Docker Compose with one container per product version. Simple, manageable, sufficient for initial rollout.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│         Host Machine / Server                │
├─────────────────────────────────────────────┤
│  Java Application (Port 8080)               │
│  ├─ Connects to database                    │
│  └─ Calls Python containers via HTTP        │
├─────────────────────────────────────────────┤
│  Python Container 1: home-loan-v1-2 (8001)  │
│  Python Container 2: home-loan-v1-3 (8002)  │
│  Python Container 3: personal-loan-v2-0(8003)│
└─────────────────────────────────────────────┘
```

---

## Docker Compose Configuration

**File**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  # Python Container: Home Loan v1.2
  python-home-loan-v1-2:
    build:
      context: ./python-modules/home_loan_v1_2
      dockerfile: Dockerfile
    container_name: home-loan-v1-2
    ports:
      - "8001:8001"
    environment:
      - PYTHON_ENGINE_TOKEN=${PYTHON_ENGINE_TOKEN}
      - LOG_LEVEL=INFO
    networks:
      - lms-internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # Python Container: Home Loan v1.3
  python-home-loan-v1-3:
    build:
      context: ./python-modules/home_loan_v1_3
      dockerfile: Dockerfile
    container_name: home-loan-v1-3
    ports:
      - "8002:8002"
    environment:
      - PYTHON_ENGINE_TOKEN=${PYTHON_ENGINE_TOKEN}
      - LOG_LEVEL=INFO
    networks:
      - lms-internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # Python Container: Personal Loan v2.0
  python-personal-loan-v2-0:
    build:
      context: ./python-modules/personal_loan_v2_0
      dockerfile: Dockerfile
    container_name: personal-loan-v2-0
    ports:
      - "8003:8003"
    environment:
      - PYTHON_ENGINE_TOKEN=${PYTHON_ENGINE_TOKEN}
      - LOG_LEVEL=INFO
    networks:
      - lms-internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8003/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  lms-internal:
    driver: bridge
```

---

## Python Container Structure

```
python-modules/
├── home_loan_v1_2/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py               # Flask/FastAPI application
│   └── home_loan_v1_2.py    # Product logic
├── home_loan_v1_3/
│   └── ...
└── personal_loan_v2_0/
    └── ...
```

---

## Dockerfile Template

**File**: `python-modules/home_loan_v1_2/Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .
COPY home_loan_v1_2.py .

# Expose port
EXPOSE 8001

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8001/health || exit 1

# Run application
CMD ["python", "app.py"]
```

---

## Python Application (Flask)

**File**: `python-modules/home_loan_v1_2/app.py`

```python
from flask import Flask, request, jsonify
import logging
import os
from home_loan_v1_2 import HomeLoanV1_2

app = Flask(__name__)
calculator = HomeLoanV1_2()

# Configure logging
logging.basicConfig(level=os.getenv('LOG_LEVEL', 'INFO'))
logger = logging.getLogger(__name__)

# Authentication token
PYTHON_ENGINE_TOKEN = os.getenv('PYTHON_ENGINE_TOKEN')

def verify_token():
    auth_header = request.headers.get('Authorization')
    if not auth_header or auth_header != f'Bearer {PYTHON_ENGINE_TOKEN}':
        return False
    return True

@app.before_request
def authenticate():
    if request.path == '/health':
        return  # Skip auth for health check
    
    if not verify_token():
        return jsonify({'error': 'Unauthorized'}), 401

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy', 'version': '1.2'}), 200

@app.route('/loan_creation/pre', methods=['POST'])
def pre_loan_creation():
    try:
        request_data = request.get_json()
        correlation_id = request.headers.get('X-Correlation-ID')
        
        logger.info(f'pre_loan_creation called',
                   extra={'correlation_id': correlation_id})
        
        result = calculator.pre_loan_creation(request_data)
        
        return jsonify(result), 200
    except Exception as e:
        logger.error(f'Error in pre_loan_creation: {e}', exc_info=True)
        return jsonify({
            'status': 'error',
            'error_type': 'TECHNICAL_ERROR',
            'errors': [{'code': 'INTERNAL_ERROR', 'message': str(e)}]
        }), 500

@app.route('/loan_creation/post', methods=['POST'])
def post_loan_creation():
    try:
        request_data = request.get_json()
        result = calculator.post_loan_creation(request_data)
        return jsonify(result), 200
    except Exception as e:
        logger.error(f'Error in post_loan_creation: {e}', exc_info=True)
        return jsonify({
            'status': 'error',
            'errors': [{'message': str(e)}]
        }), 500

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
        logger.error(f'Error: {e}', exc_info=True)
        return jsonify({'status': 'error', 'errors': [str(e)]}), 500

@app.route('/payment_posting/post', methods=['POST'])
def post_payment_posting():
    try:
        data = request.get_json()
        result = calculator.post_payment_posting(
            data['loan_data'],
            data['payment_data'],
            data.get('transaction_data', {})
        )
        return jsonify(result), 200
    except Exception as e:
        logger.error(f'Error: {e}', exc_info=True)
        return jsonify({'status': 'error', 'errors': [str(e)]}), 500

if __name__ == '__main__':
    port = int(os.getenv('PORT', '8001'))
    app.run(host='0.0.0.0', port=port)
```

---

## Dependencies

**File**: `requirements.txt`

```
Flask==3.0.0
python-dateutil==2.8.2
```

---

## Environment Configuration

**File**: `.env`

```bash
# Python Engine Authentication
PYTHON_ENGINE_TOKEN=your-secure-token-here

# Logging
LOG_LEVEL=INFO

# Container Mapping (for Java application)
PYTHON_HOME_LOAN_V1_2_URL=http://home-loan-v1-2:8001
PYTHON_HOME_LOAN_V1_3_URL=http://home-loan-v1-3:8002
PYTHON_PERSONAL_LOAN_V2_0_URL=http://personal-loan-v2-0:8003
```

---

## Java Configuration

**File**: `application.yml`

```yaml
python:
  engine:
    token: ${PYTHON_ENGINE_TOKEN}
    timeout:
      connect: 5s
      read: 30s
    retry:
      max-attempts: 3
      initial-delay: 100ms
      multiplier: 2.0
      max-delay: 2s
    containers:
      HOME_LOAN:
        "1.2": ${PYTHON_HOME_LOAN_V1_2_URL}
        "1.3": ${PYTHON_HOME_LOAN_V1_3_URL}
      PERSONAL_LOAN:
        "2.0": ${PYTHON_PERSONAL_LOAN_V2_0_URL}
```

---

## Deployment Steps

### 1. Build Python Containers

```bash
# Build all containers
docker-compose build

# Or build specific container
docker-compose build python-home-loan-v1-2
```

### 2. Start Containers

```bash
# Start all containers
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f python-home-loan-v1-2
```

### 3. Verify Health

```bash
# Check health endpoints
curl http://localhost:8001/health
curl http://localhost:8002/health
curl http://localhost:8003/health

# Expected response:
# {"status": "healthy", "version": "1.2"}
```

### 4. Test Integration

```bash
# Test pre-hook (replace token)
curl -X POST http://localhost:8001/loan_creation/pre \
  -H "Authorization: Bearer your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "request_id": "test-123",
    "loan_data": {
      "principal_amount": 1000000,
      "tenure": 120,
      "interest_rate": 8.5
    }
  }'
```

---

## Monitoring

### Container Logs

```bash
# View logs for specific container
docker logs -f home-loan-v1-2

# View logs for all Python containers
docker-compose logs -f | grep python
```

### Resource Usage

```bash
# Container stats
docker stats

# Expected: Each container ~50-100MB RAM
```

---

## Scaling

### Add New Product Version

1. Create directory: `python-modules/home_loan_v1_4/`
2. Add files: Dockerfile, app.py, home_loan_v1_4.py
3. Update `docker-compose.yml`:
   ```yaml
   python-home-loan-v1-4:
     build: ./python-modules/home_loan_v1_4
     ports: ["8004:8004"]
   ```
4. Update Java configuration
5. Build and deploy: `docker-compose up -d python-home-loan-v1-4`

---

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker logs home-loan-v1-2

# Common issues:
# - Port already in use
# - Missing environment variables
# - Python syntax errors
```

### Java Can't Connect

```bash
# Test from Java host
curl http://home-loan-v1-2:8001/health

# If fails:
# - Check network configuration
# - Verify container is running
# - Check firewall rules
```

---

## Security Checklist

- [ ] Set strong PYTHON_ENGINE_TOKEN
- [ ] Python containers only on internal network
- [ ] No external port exposure for Python containers
- [ ] Enable HTTPS for production (nginx reverse proxy)
- [ ] Regular security updates for base images

---

## Related Documents

- [Architecture Comparison](../06-technical-design/architecture-comparison-java-python.md)
- [Java-Python Integration](../06-technical-design/java-python-integration.md)
- [Python Module Specification](../06-technical-design/python-module-specification.md)

---

## Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-04 | 1.0 | Initial deployment guide for Phase 1 |
