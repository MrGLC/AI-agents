# Project 07: Deploy Ensemble Model with Load Balancing

## Overview

Deploy an ensemble of machine learning models with NGINX load balancing, implementing voting strategies, weighted ensembles, and horizontal scaling. This project demonstrates how to combine multiple models for better predictions and handle high traffic with distributed model serving.

## Learning Objectives

- Create ensemble models (voting, stacking, blending)
- Deploy multiple model instances with Docker
- Configure NGINX for load balancing ML APIs
- Implement weighted voting strategies
- Handle model disagreements
- Monitor ensemble performance
- Scale horizontally with multiple workers
- Implement health checks and failover
- Optimize latency vs accuracy tradeoffs

## Difficulty Level

**Advanced** - Requires understanding of ensemble methods, load balancing, and distributed systems.

## Technical Stack

- **ML Frameworks**: scikit-learn, XGBoost, LightGBM
- **Web Framework**: FastAPI
- **Load Balancer**: NGINX
- **Container**: Docker, Docker Compose
- **Monitoring**: Prometheus, Grafana
- **Caching**: Redis
- **Consensus**: Voting algorithms

## Requirements

### Model Requirements
- Multiple diverse base models
- Ensemble combination strategies
- Model weight optimization
- Prediction aggregation logic

### Load Balancing Requirements
- NGINX reverse proxy
- Round-robin distribution
- Health check endpoints
- Sticky sessions (optional)
- Failover handling

### API Requirements
- POST /predict - Ensemble prediction
- POST /predict/individual - Individual model predictions
- GET /models - List all models in ensemble
- GET /health - Health check
- GET /metrics - Prometheus metrics

## Step-by-Step Implementation

### Step 1: Train Ensemble Models

Create `train_ensemble.py`:

```python
import numpy as np
import pandas as pd
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
import xgboost as xgb
import lightgbm as lgb
import joblib
import json
from datetime import datetime
import os

def create_dataset():
    """Create classification dataset"""
    X, y = make_classification(
        n_samples=10000,
        n_features=20,
        n_informative=15,
        n_redundant=5,
        n_classes=2,
        weights=[0.6, 0.4],
        random_state=42
    )

    return train_test_split(X, y, test_size=0.2, random_state=42)

def train_ensemble():
    """Train multiple diverse models"""

    print("Creating dataset...")
    X_train, X_test, y_train, y_test = create_dataset()

    models = {}

    # Model 1: Random Forest
    print("\nTraining Random Forest...")
    rf = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)
    rf.fit(X_train, y_train)
    rf_acc = rf.score(X_test, y_test)
    print(f"Random Forest Accuracy: {rf_acc:.4f}")
    models['random_forest'] = {'model': rf, 'accuracy': rf_acc, 'weight': rf_acc}

    # Model 2: Gradient Boosting
    print("\nTraining Gradient Boosting...")
    gb = GradientBoostingClassifier(n_estimators=100, max_depth=5, random_state=42)
    gb.fit(X_train, y_train)
    gb_acc = gb.score(X_test, y_test)
    print(f"Gradient Boosting Accuracy: {gb_acc:.4f}")
    models['gradient_boosting'] = {'model': gb, 'accuracy': gb_acc, 'weight': gb_acc}

    # Model 3: XGBoost
    print("\nTraining XGBoost...")
    xgb_model = xgb.XGBClassifier(n_estimators=100, max_depth=6, random_state=42)
    xgb_model.fit(X_train, y_train)
    xgb_acc = xgb_model.score(X_test, y_test)
    print(f"XGBoost Accuracy: {xgb_acc:.4f}")
    models['xgboost'] = {'model': xgb_model, 'accuracy': xgb_acc, 'weight': xgb_acc}

    # Model 4: LightGBM
    print("\nTraining LightGBM...")
    lgb_model = lgb.LGBMClassifier(n_estimators=100, max_depth=6, random_state=42)
    lgb_model.fit(X_train, y_train)
    lgb_acc = lgb_model.score(X_test, y_test)
    print(f"LightGBM Accuracy: {lgb_acc:.4f}")
    models['lightgbm'] = {'model': lgb_model, 'accuracy': lgb_acc, 'weight': lgb_acc}

    # Model 5: Logistic Regression
    print("\nTraining Logistic Regression...")
    lr = LogisticRegression(max_iter=1000, random_state=42)
    lr.fit(X_train, y_train)
    lr_acc = lr.score(X_test, y_test)
    print(f"Logistic Regression Accuracy: {lr_acc:.4f}")
    models['logistic_regression'] = {'model': lr, 'accuracy': lr_acc, 'weight': lr_acc}

    # Test ensemble
    print("\n" + "="*60)
    print("Testing Ensemble Performance")
    print("="*60)

    # Simple voting
    predictions = []
    for name, model_info in models.items():
        pred = model_info['model'].predict(X_test)
        predictions.append(pred)

    # Majority vote
    vote_predictions = np.round(np.mean(predictions, axis=0))
    vote_acc = (vote_predictions == y_test).mean()
    print(f"Simple Voting Accuracy: {vote_acc:.4f}")

    # Weighted voting
    weights = np.array([m['weight'] for m in models.values()])
    weights = weights / weights.sum()

    weighted_preds = np.zeros(len(y_test))
    for i, (name, model_info) in enumerate(models.items()):
        pred_proba = model_info['model'].predict_proba(X_test)[:, 1]
        weighted_preds += weights[i] * pred_proba

    weighted_predictions = (weighted_preds > 0.5).astype(int)
    weighted_acc = (weighted_predictions == y_test).mean()
    print(f"Weighted Voting Accuracy: {weighted_acc:.4f}")

    # Save models
    os.makedirs('models', exist_ok=True)
    for name, model_info in models.items():
        model_path = f'models/{name}.joblib'
        joblib.dump(model_info['model'], model_path)
        print(f"Saved {name} to {model_path}")

    # Save ensemble metadata
    metadata = {
        'models': {
            name: {
                'accuracy': info['accuracy'],
                'weight': info['weight'] / sum(m['weight'] for m in models.values())
            }
            for name, info in models.items()
        },
        'ensemble_performance': {
            'simple_voting': float(vote_acc),
            'weighted_voting': float(weighted_acc)
        },
        'n_features': 20,
        'created_at': datetime.now().isoformat()
    }

    with open('models/ensemble_metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"\nMetadata saved to models/ensemble_metadata.json")

    return models, metadata

if __name__ == "__main__":
    train_ensemble()
```

### Step 2: Create Model Service

Create `model_service.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Dict, Optional
import joblib
import numpy as np
import json
import logging
from datetime import datetime
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Service configuration
SERVICE_ID = os.environ.get('SERVICE_ID', '1')
PORT = int(os.environ.get('PORT', 8000))

app = FastAPI(
    title=f"Ensemble Model Service {SERVICE_ID}",
    description="Individual service in ensemble deployment",
    version="1.0.0"
)

# Global variables
models = {}
metadata = {}

class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=20, max_items=20)

class ModelPrediction(BaseModel):
    model_name: str
    prediction: int
    probability: float
    confidence: float

class EnsemblePredictionResponse(BaseModel):
    prediction: int
    probability: float
    voting_strategy: str
    individual_predictions: List[ModelPrediction]
    service_id: str
    timestamp: str

@app.on_event("startup")
async def load_models():
    """Load all ensemble models"""
    global models, metadata

    try:
        # Load metadata
        with open('models/ensemble_metadata.json', 'r') as f:
            metadata = json.load(f)

        # Load each model
        for model_name in metadata['models'].keys():
            model_path = f'models/{model_name}.joblib'
            models[model_name] = joblib.load(model_path)
            logger.info(f"Loaded {model_name}")

        logger.info(f"Service {SERVICE_ID}: All models loaded successfully")

    except Exception as e:
        logger.error(f"Failed to load models: {e}")
        raise

def simple_voting(predictions: List[ModelPrediction]) -> tuple:
    """Simple majority voting"""
    votes = [p.prediction for p in predictions]
    prediction = int(np.round(np.mean(votes)))
    probability = np.mean([p.probability for p in predictions])
    return prediction, probability

def weighted_voting(predictions: List[ModelPrediction]) -> tuple:
    """Weighted voting based on model accuracy"""
    weighted_prob = 0.0
    total_weight = 0.0

    for pred in predictions:
        weight = metadata['models'][pred.model_name]['weight']
        weighted_prob += weight * pred.probability
        total_weight += weight

    probability = weighted_prob / total_weight if total_weight > 0 else 0.5
    prediction = int(probability > 0.5)

    return prediction, probability

def confidence_weighted_voting(predictions: List[ModelPrediction]) -> tuple:
    """Voting weighted by prediction confidence"""
    weighted_prob = 0.0
    total_confidence = 0.0

    for pred in predictions:
        weighted_prob += pred.confidence * pred.probability
        total_confidence += pred.confidence

    probability = weighted_prob / total_confidence if total_confidence > 0 else 0.5
    prediction = int(probability > 0.5)

    return prediction, probability

@app.get("/")
def root():
    """Root endpoint"""
    return {
        "service": f"Ensemble Model Service {SERVICE_ID}",
        "models": list(models.keys()),
        "port": PORT
    }

@app.get("/health")
def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "service_id": SERVICE_ID,
        "models_loaded": len(models),
        "timestamp": datetime.now().isoformat()
    }

@app.get("/models")
def list_models():
    """List all models with metadata"""
    return {
        "models": metadata['models'],
        "count": len(models),
        "service_id": SERVICE_ID
    }

@app.post("/predict", response_model=EnsemblePredictionResponse)
async def predict_ensemble(
    input_data: PredictionInput,
    strategy: str = "weighted"
):
    """Ensemble prediction with configurable strategy"""

    if not models:
        raise HTTPException(status_code=503, detail="Models not loaded")

    try:
        features = np.array(input_data.features).reshape(1, -1)
        individual_predictions = []

        # Get predictions from all models
        for model_name, model in models.items():
            pred_proba = model.predict_proba(features)[0]
            prediction = int(np.argmax(pred_proba))
            probability = float(pred_proba[1])
            confidence = float(max(pred_proba))

            individual_predictions.append(ModelPrediction(
                model_name=model_name,
                prediction=prediction,
                probability=probability,
                confidence=confidence
            ))

        # Apply voting strategy
        if strategy == "simple":
            final_pred, final_prob = simple_voting(individual_predictions)
        elif strategy == "weighted":
            final_pred, final_prob = weighted_voting(individual_predictions)
        elif strategy == "confidence":
            final_pred, final_prob = confidence_weighted_voting(individual_predictions)
        else:
            raise HTTPException(status_code=400, detail=f"Unknown strategy: {strategy}")

        return EnsemblePredictionResponse(
            prediction=final_pred,
            probability=final_prob,
            voting_strategy=strategy,
            individual_predictions=individual_predictions,
            service_id=SERVICE_ID,
            timestamp=datetime.now().isoformat()
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/individual")
async def predict_individual(input_data: PredictionInput, model_name: str):
    """Get prediction from specific model"""

    if model_name not in models:
        raise HTTPException(status_code=404, detail=f"Model {model_name} not found")

    try:
        features = np.array(input_data.features).reshape(1, -1)
        model = models[model_name]

        pred_proba = model.predict_proba(features)[0]
        prediction = int(np.argmax(pred_proba))
        probability = float(pred_proba[1])

        return {
            'model_name': model_name,
            'prediction': prediction,
            'probability': probability,
            'service_id': SERVICE_ID,
            'timestamp': datetime.now().isoformat()
        }

    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=PORT)
```

### Step 3: Create NGINX Configuration

Create `nginx.conf`:

```nginx
upstream ensemble_backend {
    # Load balancing strategy
    least_conn;  # Route to least busy server

    # Backend servers
    server service1:8001 max_fails=3 fail_timeout=30s;
    server service2:8002 max_fails=3 fail_timeout=30s;
    server service3:8003 max_fails=3 fail_timeout=30s;

    # Health check
    keepalive 32;
}

server {
    listen 80;
    server_name localhost;

    # Increase timeouts for ML inference
    proxy_connect_timeout 120s;
    proxy_send_timeout 120s;
    proxy_read_timeout 120s;

    # Logging
    access_log /var/log/nginx/ensemble_access.log;
    error_log /var/log/nginx/ensemble_error.log;

    # Health check endpoint
    location /health {
        proxy_pass http://ensemble_backend/health;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Main prediction endpoint
    location /predict {
        proxy_pass http://ensemble_backend/predict;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Service-Start-Time $msec;

        # Add response headers
        add_header X-Served-By $upstream_addr always;
        add_header X-Response-Time $upstream_response_time always;
    }

    # Other endpoints
    location / {
        proxy_pass http://ensemble_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Metrics endpoint
    location /metrics {
        proxy_pass http://ensemble_backend/metrics;
    }
}
```

### Step 4: Create Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # Ensemble service instances
  service1:
    build: .
    container_name: ensemble-service-1
    environment:
      - SERVICE_ID=1
      - PORT=8001
    ports:
      - "8001:8001"
    volumes:
      - ./models:/app/models
    command: uvicorn model_service:app --host 0.0.0.0 --port 8001
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  service2:
    build: .
    container_name: ensemble-service-2
    environment:
      - SERVICE_ID=2
      - PORT=8002
    ports:
      - "8002:8002"
    volumes:
      - ./models:/app/models
    command: uvicorn model_service:app --host 0.0.0.0 --port 8002
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  service3:
    build: .
    container_name: ensemble-service-3
    environment:
      - SERVICE_ID=3
      - PORT=8003
    ports:
      - "8003:8003"
    volumes:
      - ./models:/app/models
    command: uvicorn model_service:app --host 0.0.0.0 --port 8003
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8003/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # NGINX load balancer
  nginx:
    image: nginx:alpine
    container_name: ensemble-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - service1
      - service2
      - service3
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis for caching (optional)
  redis:
    image: redis:alpine
    container_name: ensemble-redis
    ports:
      - "6379:6379"
    restart: unless-stopped

  # Prometheus for monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: ensemble-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped
```

### Step 5: Create Requirements and Dockerfile

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
xgboost==2.0.3
lightgbm==4.1.0
numpy==1.24.3
joblib==1.3.2
redis==5.0.1
prometheus-client==0.19.0
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY model_service.py .
COPY models/ models/

EXPOSE 8001 8002 8003

CMD ["uvicorn", "model_service:app", "--host", "0.0.0.0", "--port", "8001"]
```

### Step 6: Create Test Suite

Create `test_ensemble.py`:

```python
import requests
import numpy as np
import time
import concurrent.futures
from collections import Counter

BASE_URL = "http://localhost"

def test_load_balancing():
    """Test load balancing across services"""

    print("Testing Load Balancing\n")

    # Generate test data
    features = np.random.rand(20).tolist()

    # Make 30 requests
    service_ids = []
    for i in range(30):
        response = requests.post(
            f"{BASE_URL}/predict",
            json={'features': features, 'strategy': 'weighted'}
        )
        result = response.json()
        service_ids.append(result['service_id'])
        print(f"Request {i+1}: Service {result['service_id']}")

    # Count distribution
    distribution = Counter(service_ids)
    print(f"\nDistribution: {dict(distribution)}")

def test_voting_strategies():
    """Test different voting strategies"""

    print("\nTesting Voting Strategies\n")

    features = np.random.rand(20).tolist()

    strategies = ['simple', 'weighted', 'confidence']

    for strategy in strategies:
        response = requests.post(
            f"{BASE_URL}/predict",
            json={'features': features, 'strategy': strategy}
        )
        result = response.json()

        print(f"\n{strategy.upper()} Voting:")
        print(f"  Prediction: {result['prediction']}")
        print(f"  Probability: {result['probability']:.4f}")
        print(f"  Service: {result['service_id']}")

        print("  Individual predictions:")
        for pred in result['individual_predictions']:
            print(f"    {pred['model_name']}: {pred['prediction']} "
                  f"(prob={pred['probability']:.4f}, conf={pred['confidence']:.4f})")

def test_concurrency():
    """Test concurrent requests"""

    print("\nTesting Concurrency\n")

    def make_request(i):
        features = np.random.rand(20).tolist()
        start = time.time()
        response = requests.post(
            f"{BASE_URL}/predict",
            json={'features': features}
        )
        duration = time.time() - start
        return duration, response.json()['service_id']

    # Make 50 concurrent requests
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(make_request, range(50)))

    durations = [r[0] for r in results]
    service_ids = [r[1] for r in results]

    print(f"Completed 50 concurrent requests")
    print(f"Average latency: {np.mean(durations)*1000:.2f}ms")
    print(f"Min: {min(durations)*1000:.2f}ms, Max: {max(durations)*1000:.2f}ms")
    print(f"Service distribution: {dict(Counter(service_ids))}")

def test_failover():
    """Test failover when a service is down"""

    print("\nTesting Failover (manual test)\n")
    print("1. Stop one service: docker stop ensemble-service-2")
    print("2. Make requests and observe distribution")
    print("3. Restart service: docker start ensemble-service-2")
    print("4. Verify requests distribute to all services again")

if __name__ == "__main__":
    test_load_balancing()
    test_voting_strategies()
    test_concurrency()
    test_failover()
```

### Step 7: Create Deployment Script

Create `deploy.sh`:

```bash
#!/bin/bash

set -e

echo "Ensemble Model Deployment with Load Balancing"
echo "=============================================="

# Train models
echo "1. Training ensemble models..."
python train_ensemble.py

# Build and start services
echo "2. Building Docker images..."
docker-compose build

echo "3. Starting all services..."
docker-compose up -d

echo "Waiting for services to be ready..."
sleep 10

# Health check
echo "4. Checking health..."
curl -s http://localhost/health | python -m json.tool

echo ""
echo "Deployment complete!"
echo "Load Balancer: http://localhost"
echo "Direct Services:"
echo "  - Service 1: http://localhost:8001"
echo "  - Service 2: http://localhost:8002"
echo "  - Service 3: http://localhost:8003"
echo "Prometheus: http://localhost:9090"
```

## Expected Outputs

### Ensemble Prediction
```json
{
  "prediction": 1,
  "probability": 0.7845,
  "voting_strategy": "weighted",
  "individual_predictions": [
    {"model_name": "random_forest", "prediction": 1, "probability": 0.75, "confidence": 0.75},
    {"model_name": "xgboost", "prediction": 1, "probability": 0.82, "confidence": 0.82},
    {"model_name": "lightgbm", "prediction": 1, "probability": 0.79, "confidence": 0.79},
    {"model_name": "gradient_boosting", "prediction": 1, "probability": 0.76, "confidence": 0.76},
    {"model_name": "logistic_regression", "prediction": 0, "probability": 0.48, "confidence": 0.52}
  ],
  "service_id": "2",
  "timestamp": "2024-11-14T10:30:00"
}
```

### Load Distribution
```
Service 1: 33% of requests
Service 2: 34% of requests
Service 3: 33% of requests
```

## Bonus Challenges

1. **Dynamic Weights**: Adjust weights based on recent performance
2. **Model Selection**: Choose subset of models based on input features
3. **Cascade Ensemble**: Use fast models first, slow models only when uncertain
4. **Online Learning**: Update model weights based on feedback
5. **Geo-Distributed**: Deploy across multiple regions
6. **Circuit Breaker**: Implement circuit breaker pattern
7. **Request Routing**: Route based on prediction confidence
8. **Model Pruning**: Remove underperforming models
9. **Uncertainty Quantification**: Measure ensemble uncertainty
10. **Cost-Aware**: Balance accuracy vs computational cost

## Resources

- [Ensemble Learning Guide](https://scikit-learn.org/stable/modules/ensemble.html)
- [NGINX Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/)
- [Voting Classifiers](https://scikit-learn.org/stable/modules/ensemble.html#voting-classifier)

## Success Criteria

- [ ] Multiple models trained successfully
- [ ] All services start and respond to health checks
- [ ] NGINX distributes requests evenly
- [ ] Different voting strategies produce different results
- [ ] Load balancer handles concurrent requests
- [ ] Failover works when service goes down
- [ ] Ensemble improves over individual models
- [ ] Response times < 200ms for ensemble
- [ ] Health checks detect unhealthy services
- [ ] Weighted voting uses accuracy weights
- [ ] Individual model predictions accessible
- [ ] Load distribution is balanced

## Project Structure

```
ensemble-load-balancing/
├── models/
│   ├── random_forest.joblib
│   ├── xgboost.joblib
│   ├── lightgbm.joblib
│   ├── gradient_boosting.joblib
│   ├── logistic_regression.joblib
│   └── ensemble_metadata.json
├── model_service.py
├── train_ensemble.py
├── test_ensemble.py
├── docker-compose.yml
├── Dockerfile
├── nginx.conf
├── requirements.txt
├── deploy.sh
└── README.md
```
