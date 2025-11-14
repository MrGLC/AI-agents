# Project 01: Deploy Scikit-Learn Model with Flask API

## Overview

Build a production-ready REST API to serve a scikit-learn machine learning model using Flask. This project demonstrates the fundamental concepts of ML model deployment including model serialization, API endpoint creation, input validation, and basic monitoring.

## Learning Objectives

- Serialize and load scikit-learn models using joblib/pickle
- Create RESTful API endpoints with Flask
- Implement input validation and error handling
- Add health check and metadata endpoints
- Structure a deployable Flask application
- Test API endpoints with realistic requests
- Implement logging and basic monitoring

## Difficulty Level

**Beginner** - This project introduces core deployment concepts with minimal infrastructure complexity.

## Technical Stack

- **ML Framework**: scikit-learn 1.3+
- **Web Framework**: Flask 3.0+
- **Model Serialization**: joblib
- **Validation**: pydantic or marshmallow
- **Testing**: pytest, requests
- **Documentation**: flask-swagger-ui or flasgger

## Requirements

### Model Requirements
- Train a classification or regression model (e.g., Random Forest, Logistic Regression)
- Save model using joblib with version tracking
- Document expected input features and their types

### API Requirements
- POST `/predict` - Make predictions
- GET `/health` - Health check endpoint
- GET `/model-info` - Return model metadata
- GET `/docs` - API documentation

### Deployment Requirements
- Request/response validation
- Proper error handling (4xx, 5xx codes)
- Logging for all requests
- CORS support for frontend integration
- Configuration via environment variables

## Step-by-Step Implementation

### Step 1: Train and Save Model

Create `train_model.py`:

```python
import pandas as pd
import numpy as np
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, accuracy_score
import joblib
import json
from datetime import datetime

def train_and_save_model():
    """Train iris classifier and save with metadata"""

    # Load dataset
    iris = load_iris()
    X, y = iris.data, iris.target

    # Split data
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    # Train model
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)

    # Evaluate
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)

    print(f"Model Accuracy: {accuracy:.4f}")
    print("\nClassification Report:")
    print(classification_report(y_test, y_pred, target_names=iris.target_names))

    # Save model
    model_path = 'models/iris_classifier.joblib'
    joblib.dump(model, model_path)

    # Save metadata
    metadata = {
        'model_type': 'RandomForestClassifier',
        'features': iris.feature_names,
        'target_names': iris.target_names.tolist(),
        'n_features': len(iris.feature_names),
        'accuracy': float(accuracy),
        'trained_at': datetime.now().isoformat(),
        'sklearn_version': '1.3.0'
    }

    with open('models/model_metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"\nModel saved to {model_path}")
    return model, metadata

if __name__ == "__main__":
    import os
    os.makedirs('models', exist_ok=True)
    train_and_save_model()
```

### Step 2: Create Flask Application

Create `app.py`:

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import joblib
import numpy as np
import json
import logging
from datetime import datetime
from typing import Dict, List
import os

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Initialize Flask app
app = Flask(__name__)
CORS(app)  # Enable CORS for all routes

# Load model and metadata at startup
MODEL_PATH = os.getenv('MODEL_PATH', 'models/iris_classifier.joblib')
METADATA_PATH = os.getenv('METADATA_PATH', 'models/model_metadata.json')

try:
    model = joblib.load(MODEL_PATH)
    with open(METADATA_PATH, 'r') as f:
        model_metadata = json.load(f)
    logger.info(f"Model loaded successfully from {MODEL_PATH}")
except Exception as e:
    logger.error(f"Failed to load model: {e}")
    model = None
    model_metadata = {}

# Input validation
def validate_input(data: Dict) -> tuple[bool, str, List]:
    """Validate prediction input"""
    if not data:
        return False, "No input data provided", None

    if 'features' not in data:
        return False, "Missing 'features' field", None

    features = data['features']

    if not isinstance(features, list):
        return False, "'features' must be a list", None

    expected_n_features = model_metadata.get('n_features', 4)
    if len(features) != expected_n_features:
        return False, f"Expected {expected_n_features} features, got {len(features)}", None

    try:
        features_array = np.array(features, dtype=float).reshape(1, -1)
        return True, "", features_array
    except ValueError as e:
        return False, f"Invalid feature values: {str(e)}", None

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    status = {
        'status': 'healthy' if model is not None else 'unhealthy',
        'timestamp': datetime.now().isoformat(),
        'model_loaded': model is not None
    }
    return jsonify(status), 200 if model else 503

@app.route('/model-info', methods=['GET'])
def model_info():
    """Return model metadata"""
    if not model_metadata:
        return jsonify({'error': 'Model metadata not available'}), 503

    return jsonify({
        'model_info': model_metadata,
        'api_version': '1.0.0'
    }), 200

@app.route('/predict', methods=['POST'])
def predict():
    """Make prediction endpoint"""
    try:
        # Check if model is loaded
        if model is None:
            logger.error("Prediction attempted with no model loaded")
            return jsonify({'error': 'Model not loaded'}), 503

        # Get request data
        data = request.get_json()
        logger.info(f"Prediction request received: {data}")

        # Validate input
        is_valid, error_msg, features_array = validate_input(data)
        if not is_valid:
            logger.warning(f"Invalid input: {error_msg}")
            return jsonify({'error': error_msg}), 400

        # Make prediction
        prediction = model.predict(features_array)
        probabilities = model.predict_proba(features_array)

        # Format response
        response = {
            'prediction': int(prediction[0]),
            'prediction_label': model_metadata['target_names'][int(prediction[0])],
            'probabilities': {
                model_metadata['target_names'][i]: float(prob)
                for i, prob in enumerate(probabilities[0])
            },
            'timestamp': datetime.now().isoformat()
        }

        logger.info(f"Prediction successful: {response}")
        return jsonify(response), 200

    except Exception as e:
        logger.error(f"Prediction error: {str(e)}", exc_info=True)
        return jsonify({'error': 'Internal server error'}), 500

@app.route('/batch-predict', methods=['POST'])
def batch_predict():
    """Batch prediction endpoint"""
    try:
        if model is None:
            return jsonify({'error': 'Model not loaded'}), 503

        data = request.get_json()
        if 'instances' not in data:
            return jsonify({'error': "Missing 'instances' field"}), 400

        instances = data['instances']
        predictions = []

        for idx, instance in enumerate(instances):
            is_valid, error_msg, features_array = validate_input({'features': instance})
            if not is_valid:
                predictions.append({
                    'index': idx,
                    'error': error_msg,
                    'prediction': None
                })
            else:
                pred = model.predict(features_array)
                prob = model.predict_proba(features_array)
                predictions.append({
                    'index': idx,
                    'prediction': int(pred[0]),
                    'prediction_label': model_metadata['target_names'][int(pred[0])],
                    'confidence': float(max(prob[0]))
                })

        return jsonify({
            'predictions': predictions,
            'count': len(predictions),
            'timestamp': datetime.now().isoformat()
        }), 200

    except Exception as e:
        logger.error(f"Batch prediction error: {str(e)}", exc_info=True)
        return jsonify({'error': 'Internal server error'}), 500

@app.errorhandler(404)
def not_found(e):
    return jsonify({'error': 'Endpoint not found'}), 404

@app.errorhandler(500)
def internal_error(e):
    return jsonify({'error': 'Internal server error'}), 500

if __name__ == '__main__':
    port = int(os.getenv('PORT', 5000))
    debug = os.getenv('DEBUG', 'False').lower() == 'true'
    app.run(host='0.0.0.0', port=port, debug=debug)
```

### Step 3: Create Requirements File

Create `requirements.txt`:

```txt
flask==3.0.0
flask-cors==4.0.0
scikit-learn==1.3.2
numpy==1.24.3
joblib==1.3.2
gunicorn==21.2.0
pytest==7.4.3
requests==2.31.0
python-dotenv==1.0.0
```

### Step 4: Create Configuration

Create `.env`:

```bash
MODEL_PATH=models/iris_classifier.joblib
METADATA_PATH=models/model_metadata.json
PORT=5000
DEBUG=False
LOG_LEVEL=INFO
```

### Step 5: Create Test Suite

Create `test_api.py`:

```python
import pytest
import json
from app import app

@pytest.fixture
def client():
    """Create test client"""
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_health_check(client):
    """Test health check endpoint"""
    response = client.get('/health')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'status' in data
    assert data['model_loaded'] == True

def test_model_info(client):
    """Test model info endpoint"""
    response = client.get('/model-info')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'model_info' in data
    assert 'features' in data['model_info']

def test_valid_prediction(client):
    """Test valid prediction"""
    payload = {
        'features': [5.1, 3.5, 1.4, 0.2]  # Sample iris features
    }
    response = client.post('/predict',
                           data=json.dumps(payload),
                           content_type='application/json')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'prediction' in data
    assert 'prediction_label' in data
    assert 'probabilities' in data

def test_invalid_prediction_missing_features(client):
    """Test prediction with missing features"""
    payload = {'features': [5.1, 3.5]}  # Only 2 features
    response = client.post('/predict',
                           data=json.dumps(payload),
                           content_type='application/json')
    assert response.status_code == 400
    data = json.loads(response.data)
    assert 'error' in data

def test_invalid_prediction_no_data(client):
    """Test prediction with no data"""
    response = client.post('/predict',
                           data=json.dumps({}),
                           content_type='application/json')
    assert response.status_code == 400

def test_batch_prediction(client):
    """Test batch prediction"""
    payload = {
        'instances': [
            [5.1, 3.5, 1.4, 0.2],
            [6.7, 3.1, 4.7, 1.5],
            [6.3, 2.9, 5.6, 1.8]
        ]
    }
    response = client.post('/batch-predict',
                           data=json.dumps(payload),
                           content_type='application/json')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'predictions' in data
    assert len(data['predictions']) == 3

def test_invalid_endpoint(client):
    """Test invalid endpoint"""
    response = client.get('/invalid')
    assert response.status_code == 404
```

### Step 6: Create Startup Script

Create `run.sh`:

```bash
#!/bin/bash

# Load environment variables
export $(cat .env | xargs)

# Train model if it doesn't exist
if [ ! -f "models/iris_classifier.joblib" ]; then
    echo "Training model..."
    python train_model.py
fi

# Run Flask app with Gunicorn for production
gunicorn --bind 0.0.0.0:$PORT --workers 4 --timeout 120 app:app
```

### Step 7: Create Client Example

Create `client_example.py`:

```python
import requests
import json

BASE_URL = "http://localhost:5000"

def test_api():
    """Test the deployed API"""

    # Test health check
    print("Testing health check...")
    response = requests.get(f"{BASE_URL}/health")
    print(f"Status: {response.status_code}")
    print(f"Response: {response.json()}\n")

    # Test model info
    print("Testing model info...")
    response = requests.get(f"{BASE_URL}/model-info")
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

    # Test prediction
    print("Testing single prediction...")
    payload = {
        'features': [5.1, 3.5, 1.4, 0.2]  # Setosa
    }
    response = requests.post(f"{BASE_URL}/predict", json=payload)
    print(f"Input: {payload}")
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

    # Test batch prediction
    print("Testing batch prediction...")
    payload = {
        'instances': [
            [5.1, 3.5, 1.4, 0.2],  # Setosa
            [6.7, 3.1, 4.7, 1.5],  # Versicolor
            [6.3, 2.9, 5.6, 1.8]   # Virginica
        ]
    }
    response = requests.post(f"{BASE_URL}/batch-predict", json=payload)
    print(f"Response: {json.dumps(response.json(), indent=2)}\n")

if __name__ == "__main__":
    test_api()
```

## Expected Outputs

### Health Check Response
```json
{
  "status": "healthy",
  "timestamp": "2024-11-14T10:30:00",
  "model_loaded": true
}
```

### Prediction Response
```json
{
  "prediction": 0,
  "prediction_label": "setosa",
  "probabilities": {
    "setosa": 0.98,
    "versicolor": 0.01,
    "virginica": 0.01
  },
  "timestamp": "2024-11-14T10:30:15"
}
```

### Model Info Response
```json
{
  "model_info": {
    "model_type": "RandomForestClassifier",
    "features": ["sepal length (cm)", "sepal width (cm)", "petal length (cm)", "petal width (cm)"],
    "target_names": ["setosa", "versicolor", "virginica"],
    "n_features": 4,
    "accuracy": 0.9667,
    "trained_at": "2024-11-14T10:00:00",
    "sklearn_version": "1.3.0"
  },
  "api_version": "1.0.0"
}
```

## Bonus Challenges

1. **Add Swagger Documentation**: Implement OpenAPI/Swagger UI using `flasgger`
2. **Rate Limiting**: Add rate limiting using `flask-limiter`
3. **Authentication**: Implement API key authentication
4. **Model Monitoring**: Log predictions to a database for monitoring data drift
5. **Caching**: Add Redis caching for frequent predictions
6. **Input Preprocessing**: Add data preprocessing pipeline serialization
7. **Multiple Models**: Support multiple model versions with endpoint versioning (v1, v2)
8. **Metrics Endpoint**: Add Prometheus metrics endpoint
9. **Async Support**: Convert to async Flask with asyncio
10. **CI/CD**: Create GitHub Actions workflow for testing and deployment

## Resources

### Documentation
- [Flask Documentation](https://flask.palletsprojects.com/)
- [Scikit-learn Model Persistence](https://scikit-learn.org/stable/model_persistence.html)
- [Gunicorn Documentation](https://docs.gunicorn.org/)

### Tutorials
- [Deploying ML Models with Flask](https://realpython.com/flask-connexion-rest-api/)
- [Production ML with scikit-learn](https://scikit-learn.org/stable/modules/model_persistence.html)

### Tools
- **Postman**: For API testing
- **Locust**: For load testing
- **pytest**: For unit testing

## Success Criteria

- [ ] Model trains and saves with metadata
- [ ] Flask app starts without errors
- [ ] All API endpoints return correct status codes
- [ ] Input validation catches malformed requests
- [ ] Predictions return correct format with probabilities
- [ ] Health check returns 200 status
- [ ] All tests pass (pytest)
- [ ] Logging captures requests and errors
- [ ] API handles 100+ requests per second
- [ ] Documentation is clear and complete
- [ ] Client example successfully communicates with API
- [ ] Error responses are informative and properly formatted

## Project Structure

```
flask-sklearn-deployment/
├── models/
│   ├── iris_classifier.joblib
│   └── model_metadata.json
├── app.py
├── train_model.py
├── test_api.py
├── client_example.py
├── requirements.txt
├── .env
├── run.sh
└── README.md
```

## Next Steps

After completing this project, you'll be ready to:
1. Add Docker containerization (see Project 02)
2. Deploy to cloud platforms (see Projects 03-05)
3. Implement model versioning (see Project 06)
4. Add A/B testing capabilities (see Project 08)
