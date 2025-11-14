# Project 05: Deploy ML Model to Google Cloud Run

## Overview

Deploy a machine learning model to Google Cloud Run, a fully managed serverless platform that automatically scales container-based applications. This project covers containerization for Cloud Run, configuring autoscaling, implementing authentication, and integrating with other GCP services.

## Learning Objectives

- Package ML models for Cloud Run deployment
- Configure Cloud Run services with proper resource limits
- Implement automatic scaling based on traffic
- Set up authentication and IAM permissions
- Integrate with Cloud Storage for model artifacts
- Monitor and log Cloud Run services
- Configure custom domains and HTTPS
- Optimize cold start performance
- Manage costs with concurrency settings

## Difficulty Level

**Intermediate** - Requires Docker knowledge and basic GCP familiarity.

## Technical Stack

- **Cloud Platform**: Google Cloud Run, Cloud Storage
- **ML Framework**: scikit-learn, FastAPI
- **Container**: Docker
- **Server**: Uvicorn
- **Monitoring**: Cloud Logging, Cloud Monitoring
- **Storage**: Google Cloud Storage
- **Auth**: Cloud IAM, API Keys

## Requirements

### Model Requirements
- Lightweight model compatible with container constraints
- Model artifacts stored in Cloud Storage
- Fast inference (< 60 seconds)
- Support for horizontal scaling

### Cloud Run Requirements
- Dockerfile optimized for Cloud Run
- Container listens on PORT environment variable
- Health check endpoint
- Memory: 512MB - 4GB
- CPU: 1-4 vCPUs
- Request timeout: < 300 seconds

### API Requirements
- POST /predict - Make predictions
- GET /health - Health check
- GET / - Service info
- Proper error handling
- JSON request/response

## Step-by-Step Implementation

### Step 1: Train and Save Model

Create `train_model.py`:

```python
import pandas as pd
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import joblib
import json
from datetime import datetime
import os

def train_model():
    """Train a classification model"""

    # Generate sample data
    X, y = make_classification(
        n_samples=5000,
        n_features=20,
        n_informative=15,
        n_redundant=5,
        n_classes=3,
        random_state=42
    )

    # Split data
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    # Create pipeline
    pipeline = Pipeline([
        ('scaler', StandardScaler()),
        ('classifier', GradientBoostingClassifier(
            n_estimators=100,
            max_depth=5,
            random_state=42
        ))
    ])

    # Train
    print("Training model...")
    pipeline.fit(X_train, y_train)

    # Evaluate
    accuracy = pipeline.score(X_test, y_test)
    print(f"Test Accuracy: {accuracy:.4f}")

    # Save model
    os.makedirs('artifacts', exist_ok=True)
    model_path = 'artifacts/model.joblib'
    joblib.dump(pipeline, model_path, compress=3)

    # Save metadata
    metadata = {
        'model_type': 'GradientBoostingClassifier',
        'n_features': 20,
        'n_classes': 3,
        'accuracy': float(accuracy),
        'trained_at': datetime.now().isoformat(),
        'sklearn_version': '1.3.2'
    }

    with open('artifacts/metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"Model saved to {model_path}")
    print(f"Model size: {os.path.getsize(model_path) / 1024:.2f} KB")

    return pipeline, metadata

if __name__ == "__main__":
    train_model()
```

### Step 2: Create FastAPI Application

Create `main.py`:

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field, validator
from typing import List, Optional
import joblib
import numpy as np
import json
import logging
import os
from datetime import datetime
from google.cloud import storage
import tempfile

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize FastAPI
app = FastAPI(
    title="ML Model API on Cloud Run",
    description="Gradient Boosting Classifier deployed on Google Cloud Run",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global variables
model = None
metadata = None

# Configuration
GCS_BUCKET = os.environ.get('GCS_BUCKET', '')
MODEL_PATH = os.environ.get('MODEL_PATH', 'model.joblib')
METADATA_PATH = os.environ.get('METADATA_PATH', 'metadata.json')

# Pydantic models
class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=20, max_items=20)

    @validator('features')
    def validate_features(cls, v):
        if len(v) != 20:
            raise ValueError('Expected exactly 20 features')
        if not all(isinstance(x, (int, float)) for x in v):
            raise ValueError('All features must be numeric')
        return v

class PredictionResponse(BaseModel):
    prediction: int
    probabilities: List[float]
    confidence: float
    timestamp: str

class BatchPredictionInput(BaseModel):
    instances: List[List[float]]

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool
    timestamp: str
    service: str = "cloud-run"

def load_from_gcs(bucket_name: str, blob_name: str, local_path: str):
    """Download file from Google Cloud Storage"""
    try:
        storage_client = storage.Client()
        bucket = storage_client.bucket(bucket_name)
        blob = bucket.blob(blob_name)
        blob.download_to_filename(local_path)
        logger.info(f"Downloaded gs://{bucket_name}/{blob_name}")
    except Exception as e:
        logger.error(f"Failed to download from GCS: {e}")
        raise

@app.on_event("startup")
async def startup_event():
    """Load model on startup"""
    global model, metadata

    try:
        if GCS_BUCKET:
            # Load from Cloud Storage
            logger.info(f"Loading model from GCS bucket: {GCS_BUCKET}")

            with tempfile.TemporaryDirectory() as temp_dir:
                # Download model
                model_local = os.path.join(temp_dir, 'model.joblib')
                load_from_gcs(GCS_BUCKET, MODEL_PATH, model_local)
                model = joblib.load(model_local)

                # Download metadata
                metadata_local = os.path.join(temp_dir, 'metadata.json')
                load_from_gcs(GCS_BUCKET, METADATA_PATH, metadata_local)
                with open(metadata_local, 'r') as f:
                    metadata = json.load(f)

        else:
            # Load from local (for testing)
            logger.info("Loading model from local filesystem")
            model = joblib.load('artifacts/model.joblib')
            with open('artifacts/metadata.json', 'r') as f:
                metadata = json.load(f)

        logger.info("Model loaded successfully")
        logger.info(f"Model type: {metadata.get('model_type')}")

    except Exception as e:
        logger.error(f"Failed to load model: {e}")
        # Don't raise - allow service to start for health checks
        model = None
        metadata = {}

@app.get("/", response_model=dict)
async def root():
    """Root endpoint with service info"""
    return {
        "service": "ML Model on Cloud Run",
        "version": "1.0.0",
        "model_type": metadata.get('model_type', 'Unknown'),
        "endpoints": {
            "predict": "/predict",
            "batch_predict": "/batch-predict",
            "health": "/health",
            "metrics": "/metrics"
        }
    }

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    return HealthResponse(
        status="healthy" if model is not None else "unhealthy",
        model_loaded=model is not None,
        timestamp=datetime.now().isoformat()
    )

@app.post("/predict", response_model=PredictionResponse)
async def predict(input_data: PredictionInput):
    """Single prediction endpoint"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    try:
        # Prepare features
        features = np.array(input_data.features).reshape(1, -1)

        # Make prediction
        prediction = model.predict(features)
        probabilities = model.predict_proba(features)

        response = PredictionResponse(
            prediction=int(prediction[0]),
            probabilities=probabilities[0].tolist(),
            confidence=float(max(probabilities[0])),
            timestamp=datetime.now().isoformat()
        )

        logger.info(f"Prediction: {response.prediction}, Confidence: {response.confidence:.4f}")
        return response

    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/batch-predict")
async def batch_predict(input_data: BatchPredictionInput):
    """Batch prediction endpoint"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    try:
        # Validate all instances have 20 features
        for idx, instance in enumerate(input_data.instances):
            if len(instance) != 20:
                raise HTTPException(
                    status_code=400,
                    detail=f"Instance {idx} has {len(instance)} features, expected 20"
                )

        # Make predictions
        features = np.array(input_data.instances)
        predictions = model.predict(features)
        probabilities = model.predict_proba(features)

        results = []
        for i, (pred, prob) in enumerate(zip(predictions, probabilities)):
            results.append({
                'index': i,
                'prediction': int(pred),
                'probabilities': prob.tolist(),
                'confidence': float(max(prob))
            })

        return {
            'predictions': results,
            'count': len(results),
            'timestamp': datetime.now().isoformat()
        }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Batch prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics")
async def metrics():
    """Basic metrics endpoint"""
    return {
        'model_info': metadata,
        'service': 'cloud-run',
        'timestamp': datetime.now().isoformat()
    }

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """Global exception handler"""
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"}
    )

if __name__ == "__main__":
    import uvicorn
    port = int(os.environ.get("PORT", 8080))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

### Step 3: Create Dockerfile for Cloud Run

Create `Dockerfile`:

```dockerfile
# Use official Python runtime as base image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY main.py .
COPY artifacts/ artifacts/

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Cloud Run expects the container to listen on PORT env variable
ENV PORT=8080

# Expose port
EXPOSE 8080

# Run the application
CMD exec uvicorn main:app --host 0.0.0.0 --port $PORT --workers 1
```

### Step 4: Create Requirements

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
numpy==1.24.3
joblib==1.3.2
google-cloud-storage==2.10.0
python-multipart==0.0.6
```

### Step 5: Create Deployment Scripts

Create `deploy.sh`:

```bash
#!/bin/bash

set -e

# Configuration
PROJECT_ID="your-gcp-project-id"
REGION="us-central1"
SERVICE_NAME="ml-model-api"
GCS_BUCKET="${PROJECT_ID}-ml-models"

echo "Deploying to Google Cloud Run..."
echo "Project: $PROJECT_ID"
echo "Region: $REGION"
echo "Service: $SERVICE_NAME"

# Step 1: Set project
gcloud config set project $PROJECT_ID

# Step 2: Enable required APIs
echo "Enabling required APIs..."
gcloud services enable \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    storage.googleapis.com

# Step 3: Create GCS bucket if it doesn't exist
echo "Creating GCS bucket..."
gsutil mb -p $PROJECT_ID -l $REGION gs://$GCS_BUCKET 2>/dev/null || echo "Bucket already exists"

# Step 4: Train and upload model
echo "Training model..."
python train_model.py

echo "Uploading model to GCS..."
gsutil cp artifacts/model.joblib gs://$GCS_BUCKET/
gsutil cp artifacts/metadata.json gs://$GCS_BUCKET/

# Step 5: Build and deploy to Cloud Run
echo "Building and deploying to Cloud Run..."
gcloud run deploy $SERVICE_NAME \
    --source . \
    --region $REGION \
    --platform managed \
    --allow-unauthenticated \
    --memory 1Gi \
    --cpu 2 \
    --timeout 300 \
    --concurrency 80 \
    --min-instances 0 \
    --max-instances 10 \
    --set-env-vars "GCS_BUCKET=$GCS_BUCKET,MODEL_PATH=model.joblib,METADATA_PATH=metadata.json"

# Step 6: Get service URL
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
    --region $REGION \
    --format 'value(status.url)')

echo ""
echo "Deployment complete!"
echo "Service URL: $SERVICE_URL"
echo ""
echo "Test with:"
echo "curl $SERVICE_URL/health"
```

### Step 6: Create IAM Configuration

Create `setup_iam.sh`:

```bash
#!/bin/bash

PROJECT_ID="your-gcp-project-id"
SERVICE_NAME="ml-model-api"
REGION="us-central1"

# Get the service account
SERVICE_ACCOUNT=$(gcloud run services describe $SERVICE_NAME \
    --region $REGION \
    --format 'value(spec.template.spec.serviceAccountName)')

echo "Service Account: $SERVICE_ACCOUNT"

# Grant Cloud Storage permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT" \
    --role="roles/storage.objectViewer"

echo "IAM permissions configured"
```

### Step 7: Create Test Client

Create `test_cloud_run.py`:

```python
import requests
import json
import time
import sys

def test_service(service_url: str):
    """Test Cloud Run service"""

    print(f"Testing Cloud Run service at: {service_url}\n")

    # Test root endpoint
    print("1. Testing root endpoint...")
    response = requests.get(f"{service_url}/")
    print(f"Status: {response.status_code}")
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

    # Test health
    print("2. Testing health check...")
    response = requests.get(f"{service_url}/health")
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

    # Test prediction
    print("3. Testing prediction...")
    payload = {
        'features': [float(i) for i in range(20)]
    }

    start_time = time.time()
    response = requests.post(f"{service_url}/predict", json=payload)
    duration = time.time() - start_time

    print(f"Status: {response.status_code}")
    print(f"Response: {json.dumps(response.json(), indent=2)}")
    print(f"Latency: {duration*1000:.2f}ms\n")

    # Test batch prediction
    print("4. Testing batch prediction...")
    batch_payload = {
        'instances': [[float(i) for i in range(20)] for _ in range(5)]
    }

    start_time = time.time()
    response = requests.post(f"{service_url}/batch-predict", json=batch_payload)
    duration = time.time() - start_time

    print(f"Status: {response.status_code}")
    result = response.json()
    print(f"Batch size: {result['count']}")
    print(f"Total time: {duration*1000:.2f}ms")
    print(f"Per prediction: {duration*1000/result['count']:.2f}ms\n")

    # Test invalid input
    print("5. Testing invalid input...")
    invalid_payload = {
        'features': [1.0, 2.0]  # Wrong number of features
    }
    response = requests.post(f"{service_url}/predict", json=invalid_payload)
    print(f"Status: {response.status_code}")
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

    # Load test
    print("6. Running load test (50 requests)...")
    latencies = []
    for i in range(50):
        start = time.time()
        response = requests.post(f"{service_url}/predict", json=payload)
        latencies.append((time.time() - start) * 1000)

        if i % 10 == 0:
            print(f"  Progress: {i}/50")

    print(f"Average latency: {sum(latencies)/len(latencies):.2f}ms")
    print(f"Min: {min(latencies):.2f}ms, Max: {max(latencies):.2f}ms")
    print(f"p95: {sorted(latencies)[int(len(latencies)*0.95)]:.2f}ms")

if __name__ == "__main__":
    if len(sys.argv) > 1:
        service_url = sys.argv[1]
    else:
        print("Usage: python test_cloud_run.py <service-url>")
        sys.exit(1)

    test_service(service_url)
```

### Step 8: Create Cloud Run Configuration

Create `service.yaml`:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ml-model-api
  labels:
    cloud.googleapis.com/location: us-central1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: '0'
        autoscaling.knative.dev/maxScale: '10'
        run.googleapis.com/cpu-throttling: 'false'
        run.googleapis.com/startup-cpu-boost: 'true'
    spec:
      containerConcurrency: 80
      timeoutSeconds: 300
      containers:
      - image: gcr.io/PROJECT_ID/ml-model-api
        ports:
        - containerPort: 8080
        env:
        - name: GCS_BUCKET
          value: PROJECT_ID-ml-models
        - name: MODEL_PATH
          value: model.joblib
        - name: METADATA_PATH
          value: metadata.json
        resources:
          limits:
            memory: 1Gi
            cpu: '2'
        startupProbe:
          httpGet:
            path: /health
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 3
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
          periodSeconds: 10
```

### Step 9: Create Monitoring Dashboard

Create `monitoring.py`:

```python
from google.cloud import monitoring_v3
import time

def create_dashboard(project_id: str, service_name: str):
    """Create Cloud Monitoring dashboard"""

    client = monitoring_v3.DashboardsServiceClient()
    project_name = f"projects/{project_id}"

    dashboard = monitoring_v3.Dashboard(
        display_name=f"{service_name} Metrics",
        dashboard_filters=[],
        grid_layout=monitoring_v3.GridLayout(
            widgets=[
                # Request count
                monitoring_v3.Widget(
                    title="Request Count",
                    xy_chart=monitoring_v3.XyChart(
                        data_sets=[
                            monitoring_v3.XyChart.DataSet(
                                time_series_query=monitoring_v3.TimeSeriesQuery(
                                    time_series_filter=monitoring_v3.TimeSeriesFilter(
                                        filter=f'resource.type="cloud_run_revision" '
                                               f'AND resource.labels.service_name="{service_name}"'
                                    )
                                )
                            )
                        ]
                    )
                ),
                # Latency
                monitoring_v3.Widget(
                    title="Request Latency",
                    xy_chart=monitoring_v3.XyChart(
                        data_sets=[
                            monitoring_v3.XyChart.DataSet(
                                time_series_query=monitoring_v3.TimeSeriesQuery(
                                    time_series_filter=monitoring_v3.TimeSeriesFilter(
                                        filter=f'resource.type="cloud_run_revision" '
                                               f'AND metric.type="run.googleapis.com/request_latencies"'
                                    )
                                )
                            )
                        ]
                    )
                ),
            ]
        )
    )

    dashboard = client.create_dashboard(
        request={"parent": project_name, "dashboard": dashboard}
    )

    print(f"Created dashboard: {dashboard.name}")

if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1:
        create_dashboard(sys.argv[1], "ml-model-api")
```

### Step 10: Create Local Testing

Create `test_local.sh`:

```bash
#!/bin/bash

# Test locally before deploying

echo "Building Docker image..."
docker build -t ml-model-api-local .

echo "Running container..."
docker run -p 8080:8080 \
    -e PORT=8080 \
    -e GCS_BUCKET="" \
    ml-model-api-local &

CONTAINER_ID=$!

sleep 5

echo "Testing endpoints..."
curl http://localhost:8080/health

echo -e "\n\nTesting prediction..."
curl -X POST http://localhost:8080/predict \
    -H "Content-Type: application/json" \
    -d '{"features": [0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19]}'

# Cleanup
docker stop $CONTAINER_ID
```

## Expected Outputs

### Deployment Success
```
Deploying service [ml-model-api] to Cloud Run service [ml-model-api] in region [us-central1]
✓ Deploying... Done.
✓ Creating Revision...
✓ Routing traffic...
Service URL: https://ml-model-api-abc123-uc.a.run.app
```

### Prediction Response
```json
{
  "prediction": 1,
  "probabilities": [0.15, 0.72, 0.13],
  "confidence": 0.72,
  "timestamp": "2024-11-14T10:30:00"
}
```

### Performance
```
Cold start: 2-4 seconds
Warm requests: 50-150ms
Auto-scaling: 0 to 10 instances
Cost: ~$0.40 per million requests
```

## Bonus Challenges

1. **Custom Domain**: Configure custom domain with SSL
2. **Authentication**: Add Cloud IAM authentication
3. **CI/CD**: Set up Cloud Build triggers
4. **Multi-Region**: Deploy to multiple regions
5. **Traffic Splitting**: A/B test with traffic splitting
6. **Cloud Armor**: Add DDoS protection
7. **Secrets**: Use Secret Manager for API keys
8. **VPC Connector**: Connect to private resources
9. **CloudCDN**: Add CDN for static content
10. **Cost Optimization**: Implement request caching

## Resources

- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Container Runtime Contract](https://cloud.google.com/run/docs/container-contract)
- [Best Practices](https://cloud.google.com/run/docs/tips)
- [Pricing Calculator](https://cloud.google.com/products/calculator)

## Success Criteria

- [ ] Model trains and uploads to Cloud Storage
- [ ] Docker image builds successfully
- [ ] Service deploys to Cloud Run
- [ ] Health check returns 200
- [ ] Predictions work correctly
- [ ] Auto-scaling functions properly
- [ ] Cold start < 5 seconds
- [ ] Warm requests < 200ms
- [ ] Service handles concurrent requests
- [ ] Logs appear in Cloud Logging
- [ ] Metrics visible in Cloud Monitoring
- [ ] Cost per million requests < $1

## Project Structure

```
google-cloud-run/
├── artifacts/
│   ├── model.joblib
│   └── metadata.json
├── main.py
├── train_model.py
├── Dockerfile
├── requirements.txt
├── service.yaml
├── deploy.sh
├── setup_iam.sh
├── test_cloud_run.py
├── test_local.sh
├── monitoring.py
└── README.md
```

## Cost Estimation

### Cloud Run Pricing (us-central1):
- CPU: $0.00002400 per vCPU-second
- Memory: $0.00000250 per GB-second
- Requests: $0.40 per million

### Example (1M requests, 200ms avg, 1GB RAM, 1 vCPU):
- CPU: $4.80
- Memory: $0.50
- Requests: $0.40
- **Total: ~$5.70/month**
