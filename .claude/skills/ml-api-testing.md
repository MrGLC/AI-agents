# ML API Testing Skill

This skill helps you test machine learning API endpoints thoroughly and systematically.

## Testing Objectives

- Verify correct predictions for valid inputs
- Test input validation and error handling
- Check response formats and schemas
- Measure performance and latency
- Test edge cases and boundary conditions
- Ensure robust error messages
- Validate authentication and authorization
- Load testing for production readiness

## When to Use This Skill

- After deploying a new ML model API
- Before releasing to production
- When updating model versions
- Debugging API issues
- Performance optimization
- Ensuring API reliability

## Test Categories

### 1. Functional Tests

Test that the API produces correct predictions:

```python
import requests
import pytest

def test_prediction_endpoint():
    url = "http://localhost:8000/predict"
    payload = {
        "features": [1.0, 2.0, 3.0, 4.0]
    }

    response = requests.post(url, json=payload)

    assert response.status_code == 200
    assert "prediction" in response.json()
    assert isinstance(response.json()["prediction"], (int, float, list))

def test_prediction_correctness():
    """Test with known input-output pairs"""
    test_cases = [
        {"input": [1, 2, 3, 4], "expected": 0.85},
        {"input": [5, 6, 7, 8], "expected": 0.15},
    ]

    for case in test_cases:
        response = requests.post(
            "http://localhost:8000/predict",
            json={"features": case["input"]}
        )
        prediction = response.json()["prediction"]
        assert abs(prediction - case["expected"]) < 0.1
```

### 2. Input Validation Tests

Test that invalid inputs are handled properly:

```python
def test_missing_features():
    response = requests.post(
        "http://localhost:8000/predict",
        json={}
    )
    assert response.status_code == 422  # Validation error

def test_wrong_feature_count():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1, 2]}  # Too few features
    )
    assert response.status_code == 422
    assert "error" in response.json()

def test_invalid_feature_types():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": ["a", "b", "c", "d"]}  # Strings instead of numbers
    )
    assert response.status_code == 422

def test_null_values():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1, None, 3, 4]}
    )
    assert response.status_code == 422
```

### 3. Edge Cases and Boundary Tests

```python
def test_zero_values():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [0, 0, 0, 0]}
    )
    assert response.status_code == 200

def test_negative_values():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [-1, -2, -3, -4]}
    )
    assert response.status_code == 200

def test_extreme_values():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1e10, 1e10, 1e10, 1e10]}
    )
    assert response.status_code in [200, 422]  # Depends on validation

def test_very_large_input():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1.0] * 10000}  # Large feature vector
    )
    # Should either process or reject gracefully
    assert response.status_code in [200, 413, 422]
```

### 4. Performance Tests

```python
import time

def test_response_time():
    """Test that API responds within acceptable time"""
    start = time.time()
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1, 2, 3, 4]}
    )
    elapsed = time.time() - start

    assert response.status_code == 200
    assert elapsed < 1.0  # Should respond within 1 second

def test_batch_prediction_performance():
    """Test batch endpoint performance"""
    batch_size = 100
    payload = {
        "instances": [
            {"features": [1, 2, 3, 4]} for _ in range(batch_size)
        ]
    }

    start = time.time()
    response = requests.post("http://localhost:8000/predict/batch", json=payload)
    elapsed = time.time() - start

    assert response.status_code == 200
    assert len(response.json()["predictions"]) == batch_size
    print(f"Batch prediction time: {elapsed:.2f}s ({batch_size/elapsed:.1f} req/s)")
```

### 5. Load Testing

```python
import concurrent.futures

def make_prediction_request():
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1, 2, 3, 4]}
    )
    return response.status_code == 200

def test_concurrent_requests():
    """Test API under concurrent load"""
    num_requests = 100

    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(make_prediction_request) for _ in range(num_requests)]
        results = [f.result() for f in concurrent.futures.as_completed(futures)]

    success_rate = sum(results) / len(results)
    assert success_rate > 0.95  # 95% success rate
```

### 6. Integration Tests

```python
def test_model_versioning():
    """Test that model version is returned"""
    response = requests.post(
        "http://localhost:8000/predict",
        json={"features": [1, 2, 3, 4], "model_version": "v1"}
    )

    assert "model_version" in response.json()
    assert response.json()["model_version"] == "v1"

def test_health_check():
    """Test health check endpoint"""
    response = requests.get("http://localhost:8000/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_model_info():
    """Test model info endpoint"""
    response = requests.get("http://localhost:8000/info")
    assert response.status_code == 200
    assert "model_name" in response.json()
    assert "version" in response.json()
```

## Using pytest for Organization

```python
# conftest.py
import pytest

@pytest.fixture
def api_url():
    return "http://localhost:8000"

@pytest.fixture
def valid_payload():
    return {"features": [1.0, 2.0, 3.0, 4.0]}

# test_api.py
def test_prediction(api_url, valid_payload):
    response = requests.post(f"{api_url}/predict", json=valid_payload)
    assert response.status_code == 200
```

## Load Testing with Locust

```python
# locustfile.py
from locust import HttpUser, task, between

class MLAPIUser(HttpUser):
    wait_time = between(1, 3)

    @task
    def predict(self):
        self.client.post("/predict", json={
            "features": [1.0, 2.0, 3.0, 4.0]
        })

# Run: locust -f locustfile.py --host=http://localhost:8000
```

## Best Practices

- **Organize tests** by functionality (unit, integration, performance)
- **Use fixtures** for common test data and setup
- **Mock external dependencies** when needed
- **Test realistic scenarios** based on production usage
- **Automate testing** in CI/CD pipeline
- **Monitor test coverage** for all endpoints
- **Document expected behaviors** in test names
- **Test error messages** are helpful and informative
- **Verify logging** works correctly
- **Test with production-like data** when possible

## CI/CD Integration

```yaml
# .github/workflows/test-api.yml
name: Test ML API

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest requests
      - name: Start API
        run: |
          python app.py &
          sleep 5
      - name: Run tests
        run: pytest tests/
```

## Tools and Libraries

- **pytest**: Test framework
- **requests**: HTTP client
- **locust**: Load testing
- **pytest-benchmark**: Performance benchmarking
- **hypothesis**: Property-based testing
- **responses**: Mock HTTP responses
- **pytest-cov**: Code coverage
