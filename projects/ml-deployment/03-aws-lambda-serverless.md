# Project 03: Serverless ML Deployment with AWS Lambda

## Overview

Deploy a machine learning model using AWS Lambda for serverless, event-driven inference. This project covers packaging ML models for Lambda, handling cold starts, managing deployment packages, and integrating with API Gateway for HTTP access.

## Learning Objectives

- Package ML models for AWS Lambda (within size constraints)
- Use Lambda Layers for dependencies
- Optimize models for serverless constraints
- Integrate Lambda with API Gateway
- Implement Lambda function handlers for ML inference
- Manage cold start optimization
- Use S3 for model storage
- Deploy with SAM or Serverless Framework
- Monitor Lambda functions with CloudWatch

## Difficulty Level

**Intermediate** - Requires AWS knowledge and understanding of serverless constraints.

## Technical Stack

- **Cloud Platform**: AWS Lambda, API Gateway, S3
- **ML Framework**: scikit-learn (lightweight models)
- **Deployment**: AWS SAM or Serverless Framework
- **Runtime**: Python 3.11
- **Storage**: S3 for model artifacts
- **Monitoring**: CloudWatch Logs and Metrics
- **IaC**: CloudFormation/SAM templates

## Requirements

### Model Requirements
- Model size optimized for Lambda (< 250MB uncompressed)
- Use lightweight libraries (scikit-learn, XGBoost)
- Model loaded from S3 or Lambda Layer
- Fast inference (< 29 seconds)

### Lambda Requirements
- Handler function for predictions
- Input validation
- Error handling
- JSON request/response format
- Environment variable configuration

### API Requirements
- POST /predict - Invoke prediction
- GET /health - Health check
- Proper CORS headers
- API key authentication (optional)

## Step-by-Step Implementation

### Step 1: Train Lightweight Model

Create `train_model.py`:

```python
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import joblib
import json
import os

def train_model():
    """Train a lightweight classification model"""

    # Generate sample data (replace with your dataset)
    X, y = make_classification(
        n_samples=1000,
        n_features=10,
        n_informative=8,
        n_redundant=2,
        n_classes=2,
        random_state=42
    )

    # Split data
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    # Create pipeline with preprocessing
    pipeline = Pipeline([
        ('scaler', StandardScaler()),
        ('classifier', RandomForestClassifier(
            n_estimators=50,  # Reduced for smaller model size
            max_depth=10,
            random_state=42
        ))
    ])

    # Train
    pipeline.fit(X_train, y_train)

    # Evaluate
    accuracy = pipeline.score(X_test, y_test)
    print(f"Model Accuracy: {accuracy:.4f}")

    # Save model
    os.makedirs('model_artifacts', exist_ok=True)
    model_path = 'model_artifacts/model.joblib'
    joblib.dump(pipeline, model_path, compress=3)

    # Check model size
    model_size_mb = os.path.getsize(model_path) / (1024 * 1024)
    print(f"Model size: {model_size_mb:.2f} MB")

    # Save metadata
    metadata = {
        'model_type': 'RandomForestClassifier',
        'n_features': 10,
        'feature_names': [f'feature_{i}' for i in range(10)],
        'accuracy': float(accuracy),
        'model_size_mb': float(model_size_mb)
    }

    with open('model_artifacts/metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    return pipeline, metadata

if __name__ == "__main__":
    train_model()
```

### Step 2: Create Lambda Handler

Create `lambda_function.py`:

```python
import json
import os
import boto3
import joblib
import numpy as np
from typing import Dict, Any
import logging

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Global variables for model caching
model = None
metadata = None
s3_client = boto3.client('s3')

# Configuration from environment variables
MODEL_BUCKET = os.environ.get('MODEL_BUCKET', '')
MODEL_KEY = os.environ.get('MODEL_KEY', 'model.joblib')
METADATA_KEY = os.environ.get('METADATA_KEY', 'metadata.json')

def load_model_from_s3():
    """Load model from S3 bucket"""
    global model, metadata

    if model is not None:
        return  # Model already loaded

    try:
        logger.info(f"Loading model from s3://{MODEL_BUCKET}/{MODEL_KEY}")

        # Download model
        model_path = '/tmp/model.joblib'
        s3_client.download_file(MODEL_BUCKET, MODEL_KEY, model_path)
        model = joblib.load(model_path)

        # Download metadata
        metadata_path = '/tmp/metadata.json'
        s3_client.download_file(MODEL_BUCKET, METADATA_KEY, metadata_path)
        with open(metadata_path, 'r') as f:
            metadata = json.load(f)

        logger.info("Model loaded successfully")

    except Exception as e:
        logger.error(f"Failed to load model: {str(e)}")
        raise

def validate_input(data: Dict) -> tuple:
    """Validate input data"""
    if 'features' not in data:
        return False, "Missing 'features' field", None

    features = data['features']

    if not isinstance(features, list):
        return False, "'features' must be a list", None

    expected_n_features = metadata.get('n_features', 10)
    if len(features) != expected_n_features:
        return False, f"Expected {expected_n_features} features, got {len(features)}", None

    try:
        features_array = np.array(features, dtype=float).reshape(1, -1)
        return True, "", features_array
    except Exception as e:
        return False, f"Invalid feature values: {str(e)}", None

def create_response(status_code: int, body: Dict) -> Dict:
    """Create API Gateway response"""
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Access-Control-Allow-Methods': 'POST, GET, OPTIONS'
        },
        'body': json.dumps(body)
    }

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict:
    """
    Lambda handler for ML predictions

    Expected input:
    {
        "features": [1.0, 2.0, 3.0, ...]
    }
    """
    logger.info(f"Received event: {json.dumps(event)}")

    # Handle OPTIONS for CORS
    if event.get('httpMethod') == 'OPTIONS':
        return create_response(200, {'message': 'OK'})

    # Handle health check
    if event.get('path') == '/health':
        return create_response(200, {
            'status': 'healthy',
            'model_loaded': model is not None
        })

    # Load model (cached after first invocation)
    try:
        load_model_from_s3()
    except Exception as e:
        return create_response(503, {
            'error': 'Model not available',
            'details': str(e)
        })

    # Parse input
    try:
        if 'body' in event:
            body = json.loads(event['body']) if isinstance(event['body'], str) else event['body']
        else:
            body = event
    except json.JSONDecodeError:
        return create_response(400, {'error': 'Invalid JSON'})

    # Validate input
    is_valid, error_msg, features_array = validate_input(body)
    if not is_valid:
        return create_response(400, {'error': error_msg})

    # Make prediction
    try:
        prediction = model.predict(features_array)
        probabilities = model.predict_proba(features_array)

        response_body = {
            'prediction': int(prediction[0]),
            'probability': float(probabilities[0][1]),
            'model_info': {
                'type': metadata.get('model_type'),
                'accuracy': metadata.get('accuracy')
            }
        }

        logger.info(f"Prediction: {response_body}")
        return create_response(200, response_body)

    except Exception as e:
        logger.error(f"Prediction error: {str(e)}")
        return create_response(500, {
            'error': 'Prediction failed',
            'details': str(e)
        })

# For local testing
if __name__ == "__main__":
    # Test event
    test_event = {
        'httpMethod': 'POST',
        'path': '/predict',
        'body': json.dumps({
            'features': [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]
        })
    }

    # Mock environment
    os.environ['MODEL_BUCKET'] = 'my-ml-models'
    os.environ['MODEL_KEY'] = 'model.joblib'

    response = lambda_handler(test_event, None)
    print(json.dumps(response, indent=2))
```

### Step 3: Create SAM Template

Create `template.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless ML Model Deployment

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: python3.11
    Environment:
      Variables:
        MODEL_BUCKET: !Ref ModelBucket
        MODEL_KEY: model.joblib
        METADATA_KEY: metadata.json

Resources:
  # S3 Bucket for model storage
  ModelBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-models-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  # Lambda Layer for dependencies
  DependenciesLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: ml-dependencies
      Description: ML dependencies (scikit-learn, numpy, joblib)
      ContentUri: dependencies/
      CompatibleRuntimes:
        - python3.11
      RetentionPolicy: Retain

  # Lambda Function
  PredictionFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub '${AWS::StackName}-prediction'
      CodeUri: src/
      Handler: lambda_function.lambda_handler
      Layers:
        - !Ref DependenciesLayer
      Policies:
        - S3ReadPolicy:
            BucketName: !Ref ModelBucket
      Events:
        Predict:
          Type: Api
          Properties:
            Path: /predict
            Method: post
            RestApiId: !Ref PredictionApi
        Health:
          Type: Api
          Properties:
            Path: /health
            Method: get
            RestApiId: !Ref PredictionApi

  # API Gateway
  PredictionApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Cors:
        AllowMethods: "'POST, GET, OPTIONS'"
        AllowHeaders: "'Content-Type'"
        AllowOrigin: "'*'"

  # CloudWatch Log Group
  PredictionLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/lambda/${PredictionFunction}'
      RetentionInDays: 7

Outputs:
  ApiEndpoint:
    Description: "API Gateway endpoint URL"
    Value: !Sub "https://${PredictionApi}.execute-api.${AWS::Region}.amazonaws.com/prod"

  ModelBucket:
    Description: "S3 bucket for models"
    Value: !Ref ModelBucket

  FunctionArn:
    Description: "Lambda Function ARN"
    Value: !GetAtt PredictionFunction.Arn
```

### Step 4: Create Requirements for Lambda Layer

Create `requirements.txt`:

```txt
scikit-learn==1.3.2
numpy==1.24.3
joblib==1.3.2
boto3==1.29.7
```

### Step 5: Create Build Script

Create `build_layer.sh`:

```bash
#!/bin/bash

# Build Lambda Layer with dependencies

echo "Building Lambda Layer..."

# Create directory structure
mkdir -p dependencies/python

# Install dependencies
pip install -r requirements.txt -t dependencies/python

# Remove unnecessary files to reduce size
cd dependencies/python
find . -type d -name "tests" -exec rm -rf {} +
find . -type d -name "__pycache__" -exec rm -rf {} +
find . -name "*.pyc" -delete
find . -name "*.pyo" -delete
find . -type d -name "*.dist-info" -exec rm -rf {} +

cd ../..

echo "Lambda Layer built successfully"
echo "Size: $(du -sh dependencies)"
```

### Step 6: Create Deploy Script

Create `deploy.sh`:

```bash
#!/bin/bash

set -e

STACK_NAME="ml-serverless-deployment"
REGION="us-east-1"

echo "Step 1: Training model..."
python train_model.py

echo "Step 2: Building Lambda Layer..."
chmod +x build_layer.sh
./build_layer.sh

echo "Step 3: Packaging application..."
mkdir -p src
cp lambda_function.py src/

sam build --template-file template.yaml

echo "Step 4: Deploying to AWS..."
sam deploy \
    --stack-name $STACK_NAME \
    --region $REGION \
    --capabilities CAPABILITY_IAM \
    --resolve-s3

# Get bucket name from stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --region $REGION \
    --query 'Stacks[0].Outputs[?OutputKey==`ModelBucket`].OutputValue' \
    --output text)

echo "Step 5: Uploading model to S3..."
aws s3 cp model_artifacts/model.joblib s3://$BUCKET_NAME/model.joblib
aws s3 cp model_artifacts/metadata.json s3://$BUCKET_NAME/metadata.json

echo "Deployment complete!"
echo "API Endpoint:"
aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --region $REGION \
    --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`].OutputValue' \
    --output text
```

### Step 7: Create Test Script

Create `test_lambda.py`:

```python
import json
import requests
import time

def test_lambda_api(api_endpoint: str):
    """Test Lambda API"""

    print(f"Testing API at: {api_endpoint}\n")

    # Test health check
    print("1. Testing health check...")
    response = requests.get(f"{api_endpoint}/health")
    print(f"Status: {response.status_code}")
    print(f"Response: {response.json()}\n")

    # Test prediction
    print("2. Testing prediction...")
    payload = {
        'features': [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]
    }

    # Warm-up request (handles cold start)
    start_time = time.time()
    response = requests.post(f"{api_endpoint}/predict", json=payload)
    cold_start_time = time.time() - start_time

    print(f"Status: {response.status_code}")
    print(f"Response: {json.dumps(response.json(), indent=2)}")
    print(f"Cold start time: {cold_start_time:.3f}s\n")

    # Subsequent request (warm Lambda)
    print("3. Testing warm Lambda...")
    start_time = time.time()
    response = requests.post(f"{api_endpoint}/predict", json=payload)
    warm_time = time.time() - start_time

    print(f"Warm request time: {warm_time:.3f}s\n")

    # Test invalid input
    print("4. Testing invalid input...")
    invalid_payload = {'features': [1.0, 2.0]}  # Wrong number of features
    response = requests.post(f"{api_endpoint}/predict", json=invalid_payload)
    print(f"Status: {response.status_code}")
    print(f"Response: {response.json()}\n")

    # Load test
    print("5. Running load test (10 requests)...")
    times = []
    for i in range(10):
        start = time.time()
        response = requests.post(f"{api_endpoint}/predict", json=payload)
        times.append(time.time() - start)

    print(f"Average response time: {sum(times)/len(times):.3f}s")
    print(f"Min: {min(times):.3f}s, Max: {max(times):.3f}s")

if __name__ == "__main__":
    # Get API endpoint from command line or CloudFormation
    import sys

    if len(sys.argv) > 1:
        api_endpoint = sys.argv[1]
    else:
        # Fetch from CloudFormation
        import boto3
        cf = boto3.client('cloudformation', region_name='us-east-1')
        response = cf.describe_stacks(StackName='ml-serverless-deployment')
        outputs = response['Stacks'][0]['Outputs']
        api_endpoint = next(o['OutputValue'] for o in outputs if o['OutputKey'] == 'ApiEndpoint')

    test_lambda_api(api_endpoint)
```

### Step 8: Create Local Testing

Create `local_test.py`:

```python
import json
from lambda_function import lambda_handler
import os

# Mock environment variables
os.environ['MODEL_BUCKET'] = 'test-bucket'
os.environ['MODEL_KEY'] = 'model.joblib'

def test_local():
    """Test Lambda function locally"""

    # Note: For local testing, modify lambda_function.py to load model from local path
    # or use LocalStack

    # Test event
    event = {
        'httpMethod': 'POST',
        'path': '/predict',
        'body': json.dumps({
            'features': [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]
        }),
        'headers': {
            'Content-Type': 'application/json'
        }
    }

    # Invoke handler
    response = lambda_handler(event, None)

    print("Response:")
    print(json.dumps(response, indent=2))

if __name__ == "__main__":
    test_local()
```

### Step 9: Create Monitoring Dashboard

Create `cloudwatch_dashboard.json`:

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", {"stat": "Sum"}],
          [".", "Errors", {"stat": "Sum"}],
          [".", "Throttles", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Lambda Invocations"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Duration", {"stat": "Average"}],
          ["...", {"stat": "Maximum"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Lambda Duration"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "ConcurrentExecutions", {"stat": "Maximum"}]
        ],
        "period": 300,
        "stat": "Maximum",
        "region": "us-east-1",
        "title": "Concurrent Executions"
      }
    }
  ]
}
```

## Expected Outputs

### Successful Deployment
```
CloudFormation stack created successfully
API Endpoint: https://abc123.execute-api.us-east-1.amazonaws.com/prod
Model uploaded to S3: ml-serverless-deployment-models-123456789012
```

### Prediction Response
```json
{
  "statusCode": 200,
  "body": {
    "prediction": 1,
    "probability": 0.87,
    "model_info": {
      "type": "RandomForestClassifier",
      "accuracy": 0.95
    }
  }
}
```

### Performance Metrics
```
Cold start: ~2-5 seconds
Warm requests: ~50-200ms
Cost: ~$0.0000002 per request (512MB, 200ms)
```

## Bonus Challenges

1. **Lambda@Edge**: Deploy model for edge inference with CloudFront
2. **Step Functions**: Create ML pipeline with multiple Lambda functions
3. **SageMaker Integration**: Use SageMaker endpoints as fallback
4. **Model Versioning**: Implement blue/green deployments
5. **Container Image**: Use Lambda container images for larger models
6. **EFS Integration**: Mount EFS for very large models
7. **Provisioned Concurrency**: Reduce cold starts with provisioned concurrency
8. **X-Ray Tracing**: Add AWS X-Ray for distributed tracing
9. **EventBridge Integration**: Trigger predictions from events
10. **Cost Optimization**: Implement caching with ElastiCache

## Resources

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Serverless ML on AWS](https://aws.amazon.com/blogs/machine-learning/)

## Success Criteria

- [ ] Model trains and is under 250MB
- [ ] Lambda function deploys successfully
- [ ] API Gateway endpoints accessible
- [ ] Health check returns 200
- [ ] Predictions return correct format
- [ ] Cold start < 5 seconds
- [ ] Warm requests < 500ms
- [ ] Model loads from S3 correctly
- [ ] CORS headers configured properly
- [ ] Error handling works correctly
- [ ] CloudWatch logs capture all events
- [ ] Cost per 1000 requests < $0.0002

## Project Structure

```
aws-lambda-serverless/
├── src/
│   └── lambda_function.py
├── dependencies/
│   └── python/
│       └── (installed packages)
├── model_artifacts/
│   ├── model.joblib
│   └── metadata.json
├── template.yaml
├── requirements.txt
├── train_model.py
├── build_layer.sh
├── deploy.sh
├── test_lambda.py
├── local_test.py
├── cloudwatch_dashboard.json
└── README.md
```

## Cost Estimation

### Monthly costs (assuming 1M requests):
- Lambda requests: $0.20
- Lambda compute (512MB, 200ms): $1.67
- API Gateway: $3.50
- S3 storage (1GB): $0.02
- **Total: ~$5.39/month**

### Free tier:
- Lambda: 1M requests/month free
- API Gateway: 1M requests/month free (first year)
