# Project 06: Deploy XGBoost Model with Version Management

## Overview

Deploy an XGBoost model with comprehensive version management, including model registry, version tracking, rollback capabilities, and A/B testing between versions. This project demonstrates production-grade model lifecycle management using MLflow for experiment tracking and model registry.

## Learning Objectives

- Train and optimize XGBoost models
- Implement model versioning with MLflow
- Create a model registry for tracking experiments
- Build APIs that support multiple model versions
- Implement version rollback and promotion
- Track model performance metrics across versions
- Compare predictions across model versions
- Automate model deployment pipelines
- Monitor model drift and performance degradation

## Difficulty Level

**Advanced** - Requires understanding of MLOps practices and model lifecycle management.

## Technical Stack

- **ML Framework**: XGBoost 2.0+
- **Model Registry**: MLflow
- **Web Framework**: FastAPI
- **Database**: PostgreSQL (MLflow backend)
- **Storage**: S3 or local filesystem
- **API**: REST with version endpoints
- **Monitoring**: Prometheus, Grafana
- **Container**: Docker, Docker Compose

## Requirements

### Model Requirements
- Multiple XGBoost model versions
- Experiment tracking with hyperparameters
- Model metadata and lineage
- Performance metrics for each version

### Versioning Requirements
- Model registry with version management
- Version promotion (staging, production)
- Rollback capabilities
- Version comparison endpoints
- Automated version deployment

### API Requirements
- GET /models - List all models and versions
- POST /predict - Predict with latest version
- POST /predict/v{version} - Predict with specific version
- POST /compare - Compare predictions across versions
- POST /promote/{version} - Promote to production
- POST /rollback - Rollback to previous version

## Step-by-Step Implementation

### Step 1: Setup MLflow

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: mlflow-postgres
    environment:
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: mlflow
      POSTGRES_DB: mlflow
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    container_name: mlflow-server
    ports:
      - "5000:5000"
    environment:
      - MLFLOW_BACKEND_STORE_URI=postgresql://mlflow:mlflow@postgres:5432/mlflow
      - MLFLOW_ARTIFACT_ROOT=/mlflow/artifacts
    volumes:
      - mlflow_data:/mlflow/artifacts
    command: >
      mlflow server
      --backend-store-uri postgresql://mlflow:mlflow@postgres:5432/mlflow
      --default-artifact-root /mlflow/artifacts
      --host 0.0.0.0
      --port 5000
    depends_on:
      - postgres
    restart: unless-stopped

  api:
    build: .
    container_name: xgboost-api
    ports:
      - "8000:8000"
    environment:
      - MLFLOW_TRACKING_URI=http://mlflow:5000
    volumes:
      - ./models:/app/models
    depends_on:
      - mlflow
    restart: unless-stopped

volumes:
  postgres_data:
  mlflow_data:
```

### Step 2: Train Models with Version Tracking

Create `train_with_mlflow.py`:

```python
import xgboost as xgb
import mlflow
import mlflow.xgboost
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
import numpy as np
import pandas as pd
from datetime import datetime
import json

# Configure MLflow
mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("xgboost-classification")

def create_dataset():
    """Create sample classification dataset"""
    X, y = make_classification(
        n_samples=10000,
        n_features=30,
        n_informative=20,
        n_redundant=5,
        n_classes=2,
        weights=[0.7, 0.3],
        random_state=42
    )

    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    return X_train, X_test, y_train, y_test

def train_model(params, X_train, y_train, X_test, y_test, version_name=None):
    """Train XGBoost model with MLflow tracking"""

    with mlflow.start_run(run_name=version_name) as run:
        # Log parameters
        mlflow.log_params(params)

        # Create DMatrix
        dtrain = xgb.DMatrix(X_train, label=y_train)
        dtest = xgb.DMatrix(X_test, label=y_test)

        # Train model
        print(f"Training model with params: {params}")
        model = xgb.train(
            params,
            dtrain,
            num_boost_round=params.get('n_estimators', 100),
            evals=[(dtrain, 'train'), (dtest, 'test')],
            early_stopping_rounds=10,
            verbose_eval=False
        )

        # Make predictions
        y_pred_proba = model.predict(dtest)
        y_pred = (y_pred_proba > 0.5).astype(int)

        # Calculate metrics
        accuracy = accuracy_score(y_test, y_pred)
        f1 = f1_score(y_test, y_pred)
        auc = roc_auc_score(y_test, y_pred_proba)

        # Log metrics
        mlflow.log_metrics({
            'accuracy': accuracy,
            'f1_score': f1,
            'auc_roc': auc,
            'n_trees': model.num_boosted_rounds()
        })

        # Log additional info
        mlflow.set_tags({
            'framework': 'xgboost',
            'task': 'binary_classification',
            'created_at': datetime.now().isoformat()
        })

        # Log model
        mlflow.xgboost.log_model(
            model,
            artifact_path="model",
            registered_model_name="xgboost-classifier"
        )

        print(f"Run ID: {run.info.run_id}")
        print(f"Accuracy: {accuracy:.4f}")
        print(f"F1 Score: {f1:.4f}")
        print(f"AUC-ROC: {auc:.4f}")

        return run.info.run_id, model, accuracy

def train_multiple_versions():
    """Train multiple model versions with different hyperparameters"""

    print("Creating dataset...")
    X_train, X_test, y_train, y_test = create_dataset()

    # Define different configurations
    configs = [
        {
            'name': 'baseline',
            'params': {
                'objective': 'binary:logistic',
                'max_depth': 3,
                'learning_rate': 0.1,
                'n_estimators': 100,
                'subsample': 0.8,
                'colsample_bytree': 0.8
            }
        },
        {
            'name': 'deep_trees',
            'params': {
                'objective': 'binary:logistic',
                'max_depth': 7,
                'learning_rate': 0.05,
                'n_estimators': 200,
                'subsample': 0.8,
                'colsample_bytree': 0.8
            }
        },
        {
            'name': 'regularized',
            'params': {
                'objective': 'binary:logistic',
                'max_depth': 5,
                'learning_rate': 0.1,
                'n_estimators': 150,
                'subsample': 0.7,
                'colsample_bytree': 0.7,
                'reg_alpha': 0.5,
                'reg_lambda': 1.0
            }
        },
        {
            'name': 'optimized',
            'params': {
                'objective': 'binary:logistic',
                'max_depth': 6,
                'learning_rate': 0.05,
                'n_estimators': 250,
                'subsample': 0.85,
                'colsample_bytree': 0.85,
                'reg_alpha': 0.3,
                'reg_lambda': 0.5,
                'min_child_weight': 3
            }
        }
    ]

    results = []
    for config in configs:
        print(f"\nTraining version: {config['name']}")
        print("=" * 60)
        run_id, model, accuracy = train_model(
            config['params'],
            X_train, y_train,
            X_test, y_test,
            version_name=config['name']
        )
        results.append({
            'name': config['name'],
            'run_id': run_id,
            'accuracy': accuracy
        })

    # Print summary
    print("\n" + "=" * 60)
    print("Training Summary")
    print("=" * 60)
    for result in results:
        print(f"{result['name']}: {result['accuracy']:.4f} (Run: {result['run_id']})")

    return results

if __name__ == "__main__":
    train_multiple_versions()
```

### Step 3: Create Model Registry Manager

Create `model_registry.py`:

```python
import mlflow
from mlflow.tracking import MlflowClient
from typing import List, Dict, Optional
import pandas as pd

class ModelRegistry:
    """Manage model versions in MLflow registry"""

    def __init__(self, tracking_uri="http://localhost:5000"):
        mlflow.set_tracking_uri(tracking_uri)
        self.client = MlflowClient()
        self.model_name = "xgboost-classifier"

    def list_versions(self) -> List[Dict]:
        """List all model versions"""
        try:
            versions = self.client.search_model_versions(f"name='{self.model_name}'")
            return [
                {
                    'version': v.version,
                    'stage': v.current_stage,
                    'run_id': v.run_id,
                    'created_at': v.creation_timestamp,
                    'description': v.description
                }
                for v in versions
            ]
        except Exception as e:
            print(f"Error listing versions: {e}")
            return []

    def get_model_by_version(self, version: int):
        """Load model by version number"""
        model_uri = f"models:/{self.model_name}/{version}"
        return mlflow.xgboost.load_model(model_uri)

    def get_model_by_stage(self, stage: str = "Production"):
        """Load model by stage (Production, Staging, Archived)"""
        model_uri = f"models:/{self.model_name}/{stage}"
        return mlflow.xgboost.load_model(model_uri)

    def promote_to_production(self, version: int):
        """Promote a model version to production"""
        # Archive current production models
        current_production = self.client.get_latest_versions(
            self.model_name,
            stages=["Production"]
        )

        for prod_model in current_production:
            self.client.transition_model_version_stage(
                name=self.model_name,
                version=prod_model.version,
                stage="Archived"
            )

        # Promote new version
        self.client.transition_model_version_stage(
            name=self.model_name,
            version=version,
            stage="Production"
        )

        print(f"Version {version} promoted to Production")

    def promote_to_staging(self, version: int):
        """Promote a model version to staging"""
        self.client.transition_model_version_stage(
            name=self.model_name,
            version=version,
            stage="Staging"
        )
        print(f"Version {version} promoted to Staging")

    def rollback(self):
        """Rollback to previous production version"""
        archived = self.client.get_latest_versions(
            self.model_name,
            stages=["Archived"]
        )

        if not archived:
            print("No archived versions to rollback to")
            return

        # Get most recent archived version
        latest_archived = max(archived, key=lambda x: x.creation_timestamp)

        # Promote back to production
        self.promote_to_production(int(latest_archived.version))
        print(f"Rolled back to version {latest_archived.version}")

    def compare_versions(self, version1: int, version2: int) -> Dict:
        """Compare metrics between two versions"""
        runs = []
        for version in [version1, version2]:
            model_version = self.client.get_model_version(self.model_name, version)
            run = self.client.get_run(model_version.run_id)
            runs.append({
                'version': version,
                'metrics': run.data.metrics,
                'params': run.data.params
            })

        return {
            'version1': runs[0],
            'version2': runs[1]
        }

    def get_version_metadata(self, version: int) -> Dict:
        """Get metadata for a specific version"""
        model_version = self.client.get_model_version(self.model_name, version)
        run = self.client.get_run(model_version.run_id)

        return {
            'version': version,
            'stage': model_version.current_stage,
            'run_id': model_version.run_id,
            'metrics': run.data.metrics,
            'params': run.data.params,
            'tags': run.data.tags,
            'created_at': model_version.creation_timestamp
        }

if __name__ == "__main__":
    # Example usage
    registry = ModelRegistry()

    print("Available versions:")
    versions = registry.list_versions()
    for v in versions:
        print(f"  Version {v['version']}: {v['stage']}")

    # Promote version 1 to production
    if versions:
        registry.promote_to_production(1)
```

### Step 4: Create Versioned API

Create `app.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional, Dict
import xgboost as xgb
import mlflow
import numpy as np
from datetime import datetime
import logging
from model_registry import ModelRegistry

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="XGBoost Versioned API",
    description="ML API with comprehensive version management",
    version="1.0.0"
)

# Initialize model registry
registry = ModelRegistry()

# Cache for loaded models
model_cache = {}

class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=30, max_items=30)

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    model_version: str
    timestamp: str

class CompareRequest(BaseModel):
    features: List[float]
    versions: List[int]

class VersionInfo(BaseModel):
    version: int
    stage: str
    run_id: str
    metrics: Dict
    params: Dict

def load_model(version: Optional[int] = None, stage: str = "Production"):
    """Load model from cache or MLflow"""
    cache_key = f"v{version}" if version else stage

    if cache_key not in model_cache:
        try:
            if version:
                model = registry.get_model_by_version(version)
            else:
                model = registry.get_model_by_stage(stage)
            model_cache[cache_key] = model
            logger.info(f"Loaded model: {cache_key}")
        except Exception as e:
            logger.error(f"Failed to load model {cache_key}: {e}")
            raise HTTPException(status_code=503, detail="Model not available")

    return model_cache[cache_key]

@app.get("/")
def root():
    """Root endpoint"""
    return {
        "service": "XGBoost Versioned API",
        "version": "1.0.0",
        "mlflow_tracking_uri": mlflow.get_tracking_uri()
    }

@app.get("/models")
def list_models():
    """List all model versions"""
    try:
        versions = registry.list_versions()
        return {
            "model_name": registry.model_name,
            "versions": versions,
            "count": len(versions)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/models/{version}", response_model=VersionInfo)
def get_model_info(version: int):
    """Get information about specific model version"""
    try:
        metadata = registry.get_version_metadata(version)
        return VersionInfo(
            version=metadata['version'],
            stage=metadata['stage'],
            run_id=metadata['run_id'],
            metrics=metadata['metrics'],
            params=metadata['params']
        )
    except Exception as e:
        raise HTTPException(status_code=404, detail=f"Version {version} not found")

@app.post("/predict", response_model=PredictionResponse)
def predict(input_data: PredictionInput, version: Optional[int] = None):
    """Make prediction with specified version or production model"""
    try:
        # Load model
        if version:
            model = load_model(version=version)
            model_version = f"v{version}"
        else:
            model = load_model(stage="Production")
            model_version = "production"

        # Prepare data
        dmatrix = xgb.DMatrix(np.array(input_data.features).reshape(1, -1))

        # Predict
        probability = float(model.predict(dmatrix)[0])
        prediction = int(probability > 0.5)

        return PredictionResponse(
            prediction=prediction,
            probability=probability,
            model_version=model_version,
            timestamp=datetime.now().isoformat()
        )

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/v{version}", response_model=PredictionResponse)
def predict_version(version: int, input_data: PredictionInput):
    """Make prediction with specific model version"""
    return predict(input_data, version=version)

@app.post("/compare")
def compare_predictions(request: CompareRequest):
    """Compare predictions across multiple versions"""
    try:
        results = []
        dmatrix = xgb.DMatrix(np.array(request.features).reshape(1, -1))

        for version in request.versions:
            try:
                model = load_model(version=version)
                probability = float(model.predict(dmatrix)[0])
                prediction = int(probability > 0.5)

                results.append({
                    'version': version,
                    'prediction': prediction,
                    'probability': probability
                })
            except Exception as e:
                results.append({
                    'version': version,
                    'error': str(e)
                })

        return {
            'comparisons': results,
            'timestamp': datetime.now().isoformat()
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/promote/{version}")
def promote_version(version: int, stage: str = "Production"):
    """Promote a version to production or staging"""
    try:
        if stage == "Production":
            registry.promote_to_production(version)
        elif stage == "Staging":
            registry.promote_to_staging(version)
        else:
            raise ValueError(f"Invalid stage: {stage}")

        # Clear cache
        model_cache.clear()

        return {
            'message': f'Version {version} promoted to {stage}',
            'timestamp': datetime.now().isoformat()
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/rollback")
def rollback_version():
    """Rollback to previous production version"""
    try:
        registry.rollback()
        model_cache.clear()

        return {
            'message': 'Rollback successful',
            'timestamp': datetime.now().isoformat()
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/compare-versions/{version1}/{version2}")
def compare_versions(version1: int, version2: int):
    """Compare metrics between two versions"""
    try:
        comparison = registry.compare_versions(version1, version2)
        return comparison
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    """Health check"""
    try:
        versions = registry.list_versions()
        return {
            'status': 'healthy',
            'mlflow_connected': True,
            'available_versions': len(versions),
            'timestamp': datetime.now().isoformat()
        }
    except:
        return {
            'status': 'unhealthy',
            'mlflow_connected': False,
            'timestamp': datetime.now().isoformat()
        }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 5: Create Requirements

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
xgboost==2.0.3
mlflow==2.9.2
scikit-learn==1.3.2
numpy==1.24.3
pandas==2.1.4
psycopg2-binary==2.9.9
prometheus-client==0.19.0
```

### Step 6: Create Dockerfile

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app.py .
COPY model_registry.py .

# Expose port
EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 7: Create Test Suite

Create `test_versioning.py`:

```python
import requests
import json
import numpy as np

BASE_URL = "http://localhost:8000"

def test_versioning_api():
    """Test version management features"""

    print("Testing XGBoost Versioning API\n")

    # 1. List models
    print("1. Listing all models...")
    response = requests.get(f"{BASE_URL}/models")
    models = response.json()
    print(f"Found {models['count']} versions")
    for v in models['versions']:
        print(f"  Version {v['version']}: {v['stage']}")

    # 2. Get model info
    if models['count'] > 0:
        version = models['versions'][0]['version']
        print(f"\n2. Getting info for version {version}...")
        response = requests.get(f"{BASE_URL}/models/{version}")
        info = response.json()
        print(f"Metrics: {info['metrics']}")

    # 3. Predict with production model
    print("\n3. Testing prediction with production model...")
    features = np.random.rand(30).tolist()
    response = requests.post(
        f"{BASE_URL}/predict",
        json={'features': features}
    )
    result = response.json()
    print(f"Prediction: {result['prediction']}")
    print(f"Probability: {result['probability']:.4f}")
    print(f"Model version: {result['model_version']}")

    # 4. Predict with specific version
    if models['count'] > 0:
        version = models['versions'][0]['version']
        print(f"\n4. Testing prediction with version {version}...")
        response = requests.post(
            f"{BASE_URL}/predict/v{version}",
            json={'features': features}
        )
        result = response.json()
        print(f"Prediction: {result['prediction']}")

    # 5. Compare versions
    if models['count'] >= 2:
        v1 = models['versions'][0]['version']
        v2 = models['versions'][1]['version']
        print(f"\n5. Comparing versions {v1} and {v2}...")
        response = requests.post(
            f"{BASE_URL}/compare",
            json={
                'features': features,
                'versions': [v1, v2]
            }
        )
        comparison = response.json()
        for comp in comparison['comparisons']:
            print(f"Version {comp['version']}: "
                  f"pred={comp['prediction']}, "
                  f"prob={comp['probability']:.4f}")

    # 6. Health check
    print("\n6. Checking health...")
    response = requests.get(f"{BASE_URL}/health")
    health = response.json()
    print(f"Status: {health['status']}")
    print(f"Available versions: {health['available_versions']}")

if __name__ == "__main__":
    test_versioning_api()
```

### Step 8: Create Deployment Script

Create `deploy.sh`:

```bash
#!/bin/bash

set -e

echo "XGBoost Model Versioning Deployment"
echo "===================================="

# Start infrastructure
echo "1. Starting MLflow and PostgreSQL..."
docker-compose up -d postgres mlflow

echo "Waiting for MLflow to be ready..."
sleep 10

# Train models
echo "2. Training multiple model versions..."
python train_with_mlflow.py

# Promote version to production
echo "3. Promoting version 1 to production..."
python -c "from model_registry import ModelRegistry; ModelRegistry().promote_to_production(1)"

# Start API
echo "4. Starting API..."
docker-compose up -d api

echo "Deployment complete!"
echo "MLflow UI: http://localhost:5000"
echo "API: http://localhost:8000"
echo "Docs: http://localhost:8000/docs"
```

## Expected Outputs

### MLflow UI
```
Experiments: 4 versions trained
Production model: Version 4 (optimized)
Staging: Version 3
Archived: Versions 1, 2
```

### Prediction Response
```json
{
  "prediction": 1,
  "probability": 0.8532,
  "model_version": "production",
  "timestamp": "2024-11-14T10:30:00"
}
```

### Version Comparison
```json
{
  "comparisons": [
    {
      "version": 1,
      "prediction": 0,
      "probability": 0.4231
    },
    {
      "version": 4,
      "prediction": 1,
      "probability": 0.8532
    }
  ]
}
```

## Bonus Challenges

1. **Auto-Promotion**: Automatically promote models based on metrics
2. **Model Drift Detection**: Monitor prediction drift over time
3. **Shadow Deployment**: Run multiple versions in parallel
4. **Performance Tracking**: Track latency across versions
5. **Feature Store**: Integrate with feature store
6. **Data Validation**: Validate inputs against training data distribution
7. **Model Explainability**: Add SHAP values for predictions
8. **Canary Releases**: Gradual traffic shifting
9. **Model Monitoring Dashboard**: Real-time metrics dashboard
10. **Automated Retraining**: Trigger retraining on performance degradation

## Resources

- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [XGBoost Documentation](https://xgboost.readthedocs.io/)
- [Model Registry Guide](https://mlflow.org/docs/latest/model-registry.html)
- [MLOps Best Practices](https://ml-ops.org/)

## Success Criteria

- [ ] MLflow server running with PostgreSQL backend
- [ ] Multiple model versions trained and registered
- [ ] Models promote to production/staging
- [ ] API serves predictions from different versions
- [ ] Version comparison works correctly
- [ ] Rollback functionality operational
- [ ] Model metadata tracked correctly
- [ ] Metrics logged for all versions
- [ ] Version promotion is atomic
- [ ] API handles version not found errors
- [ ] MLflow UI accessible and functional
- [ ] Model cache improves performance

## Project Structure

```
xgboost-versioning/
├── models/
├── app.py
├── model_registry.py
├── train_with_mlflow.py
├── test_versioning.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── deploy.sh
└── README.md
```
