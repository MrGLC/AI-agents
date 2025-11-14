# Project 04: Deploy TensorFlow Model with TensorFlow Serving

## Overview

Deploy TensorFlow models using TensorFlow Serving, Google's high-performance serving system designed for production ML environments. This project covers model export in SavedModel format, configuring TF Serving, REST and gRPC APIs, model versioning, and batching optimization.

## Learning Objectives

- Export TensorFlow models in SavedModel format
- Set up TensorFlow Serving with Docker
- Configure model versioning and hot-swapping
- Use both REST and gRPC APIs for inference
- Implement batch inference for throughput optimization
- Monitor serving performance
- Handle multiple model versions simultaneously
- Optimize serving configuration for production

## Difficulty Level

**Intermediate to Advanced** - Requires TensorFlow knowledge and understanding of serving systems.

## Technical Stack

- **ML Framework**: TensorFlow 2.15+, Keras
- **Serving**: TensorFlow Serving
- **Container**: Docker
- **APIs**: REST (HTTP) and gRPC
- **Client**: Python, requests, grpcio
- **Monitoring**: Prometheus, TensorBoard
- **Load Balancing**: NGINX (optional)

## Requirements

### Model Requirements
- Train TensorFlow/Keras model
- Export in SavedModel format with signatures
- Support versioning (multiple model versions)
- Proper input/output specifications

### Serving Requirements
- TensorFlow Serving via Docker
- REST API on port 8501
- gRPC API on port 8500
- Model versioning support
- Batching configuration
- Health check endpoints

### API Requirements
- POST /v1/models/model_name:predict
- GET /v1/models/model_name/metadata
- GET /v1/models/model_name/versions
- Model hot-swapping without downtime

## Step-by-Step Implementation

### Step 1: Train and Export TensorFlow Model

Create `train_and_export.py`:

```python
import tensorflow as tf
from tensorflow import keras
import numpy as np
import os
import json
from datetime import datetime

def create_model():
    """Create a simple CNN for MNIST"""
    model = keras.Sequential([
        keras.layers.Input(shape=(28, 28, 1)),
        keras.layers.Conv2D(32, 3, activation='relu'),
        keras.layers.MaxPooling2D(),
        keras.layers.Conv2D(64, 3, activation='relu'),
        keras.layers.MaxPooling2D(),
        keras.layers.Flatten(),
        keras.layers.Dense(128, activation='relu'),
        keras.layers.Dropout(0.5),
        keras.layers.Dense(10, activation='softmax')
    ])
    return model

def train_model():
    """Train MNIST classifier"""
    # Load data
    (x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

    # Normalize
    x_train = x_train.astype('float32') / 255.0
    x_test = x_test.astype('float32') / 255.0

    # Reshape for CNN
    x_train = x_train.reshape(-1, 28, 28, 1)
    x_test = x_test.reshape(-1, 28, 28, 1)

    # Create and compile model
    model = create_model()
    model.compile(
        optimizer='adam',
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

    # Train
    print("Training model...")
    history = model.fit(
        x_train, y_train,
        batch_size=128,
        epochs=5,
        validation_split=0.1,
        verbose=1
    )

    # Evaluate
    test_loss, test_acc = model.evaluate(x_test, y_test, verbose=0)
    print(f"\nTest accuracy: {test_acc:.4f}")

    return model, test_acc

def export_model(model, version=1):
    """Export model in SavedModel format"""

    export_path = f'models/mnist_model/{version}'
    print(f"\nExporting model to: {export_path}")

    # Define the serving signature
    @tf.function(input_signature=[tf.TensorSpec(shape=[None, 28, 28, 1], dtype=tf.float32)])
    def serve_predict(input_tensor):
        """Serving signature for prediction"""
        predictions = model(input_tensor, training=False)
        return {
            'predictions': predictions,
            'classes': tf.argmax(predictions, axis=1)
        }

    # Save model with signature
    tf.saved_model.save(
        model,
        export_path,
        signatures={
            'serving_default': serve_predict,
            'predict': serve_predict
        }
    )

    print(f"Model exported to {export_path}")

    # Verify the export
    print("\nVerifying export...")
    loaded = tf.saved_model.load(export_path)
    print("Signatures:", list(loaded.signatures.keys()))

    # Test the loaded model
    test_input = tf.random.normal([1, 28, 28, 1])
    output = loaded.signatures['serving_default'](test_input)
    print(f"Test output shape: {output['predictions'].shape}")

    return export_path

def save_metadata(version, accuracy):
    """Save model metadata"""
    metadata = {
        'version': version,
        'framework': 'tensorflow',
        'model_type': 'CNN',
        'task': 'image_classification',
        'input_shape': [28, 28, 1],
        'output_classes': 10,
        'class_names': [str(i) for i in range(10)],
        'accuracy': float(accuracy),
        'created_at': datetime.now().isoformat(),
        'tensorflow_version': tf.__version__
    }

    metadata_path = f'models/mnist_model/{version}/metadata.json'
    with open(metadata_path, 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"Metadata saved to {metadata_path}")

def main():
    """Main training and export pipeline"""
    # Train model
    model, accuracy = train_model()

    # Export version 1
    version = 1
    export_model(model, version=version)
    save_metadata(version, accuracy)

    print("\n" + "="*50)
    print("Model ready for TensorFlow Serving!")
    print("="*50)

if __name__ == "__main__":
    main()
```

### Step 2: Create TensorFlow Serving Configuration

Create `model_config.config`:

```protobuf
model_config_list {
  config {
    name: 'mnist_model'
    base_path: '/models/mnist_model'
    model_platform: 'tensorflow'
    model_version_policy {
      all: {}
    }
  }
}
```

### Step 3: Create Batching Configuration

Create `batching_config.txt`:

```
max_batch_size { value: 128 }
batch_timeout_micros { value: 1000 }
max_enqueued_batches { value: 1000000 }
num_batch_threads { value: 8 }
```

### Step 4: Create Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  tensorflow-serving:
    image: tensorflow/serving:latest
    container_name: tf-serving
    ports:
      - "8500:8500"  # gRPC
      - "8501:8501"  # REST
    volumes:
      - ./models:/models
      - ./model_config.config:/models/model_config.config
      - ./batching_config.txt:/models/batching_config.txt
    environment:
      - MODEL_NAME=mnist_model
    command:
      - "--model_config_file=/models/model_config.config"
      - "--enable_batching=true"
      - "--batching_parameters_file=/models/batching_config.txt"
      - "--rest_api_port=8501"
      - "--allow_version_labels_for_unavailable_models=true"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8501/v1/models/mnist_model"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Optional: NGINX for load balancing multiple TF Serving instances
  nginx:
    image: nginx:alpine
    container_name: tf-nginx
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - tensorflow-serving
    restart: unless-stopped

  # Optional: Prometheus for monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: tf-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped
```

### Step 5: Create REST API Client

Create `rest_client.py`:

```python
import requests
import numpy as np
import json
from typing import Dict, List
import time

class TFServingRESTClient:
    """Client for TensorFlow Serving REST API"""

    def __init__(self, host='localhost', port=8501, model_name='mnist_model'):
        self.base_url = f"http://{host}:{port}/v1/models/{model_name}"
        self.model_name = model_name

    def get_model_status(self):
        """Get model status"""
        url = self.base_url
        response = requests.get(url)
        return response.json()

    def get_model_metadata(self, version=None):
        """Get model metadata"""
        url = f"{self.base_url}/metadata" if version is None else f"{self.base_url}/versions/{version}/metadata"
        response = requests.get(url)
        return response.json()

    def predict(self, instances: np.ndarray, version=None):
        """Make prediction"""
        url = f"{self.base_url}:predict" if version is None else f"{self.base_url}/versions/{version}:predict"

        # Prepare data
        data = {
            "instances": instances.tolist()
        }

        # Make request
        response = requests.post(url, json=data)
        return response.json()

    def predict_batch(self, instances: List[np.ndarray]):
        """Batch prediction"""
        data = {
            "instances": [img.tolist() for img in instances]
        }

        url = f"{self.base_url}:predict"
        response = requests.post(url, json=data)
        return response.json()

def test_rest_client():
    """Test REST API client"""
    client = TFServingRESTClient()

    print("1. Getting model status...")
    status = client.get_model_status()
    print(json.dumps(status, indent=2))

    print("\n2. Getting model metadata...")
    metadata = client.get_model_metadata()
    print(json.dumps(metadata, indent=2))

    print("\n3. Making single prediction...")
    # Create random test image
    test_image = np.random.rand(1, 28, 28, 1).astype(np.float32)

    start_time = time.time()
    result = client.predict(test_image)
    duration = time.time() - start_time

    print(f"Prediction: {result}")
    print(f"Time: {duration*1000:.2f}ms")

    print("\n4. Making batch prediction...")
    batch_images = np.random.rand(10, 28, 28, 1).astype(np.float32)

    start_time = time.time()
    result = client.predict_batch(batch_images)
    duration = time.time() - start_time

    print(f"Batch size: 10")
    print(f"Time: {duration*1000:.2f}ms")
    print(f"Time per image: {duration*1000/10:.2f}ms")

if __name__ == "__main__":
    test_rest_client()
```

### Step 6: Create gRPC Client

Create `grpc_client.py`:

```python
import grpc
import tensorflow as tf
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import prediction_service_pb2_grpc
import numpy as np
import time

class TFServingGRPCClient:
    """Client for TensorFlow Serving gRPC API"""

    def __init__(self, host='localhost', port=8500, model_name='mnist_model'):
        self.host = host
        self.port = port
        self.model_name = model_name
        self.channel = grpc.insecure_channel(f'{host}:{port}')
        self.stub = prediction_service_pb2_grpc.PredictionServiceStub(self.channel)

    def predict(self, instances: np.ndarray, version=None):
        """Make prediction via gRPC"""
        # Create request
        request = predict_pb2.PredictRequest()
        request.model_spec.name = self.model_name

        if version is not None:
            request.model_spec.version.value = version

        request.model_spec.signature_name = 'serving_default'

        # Set input
        request.inputs['input_1'].CopyFrom(
            tf.make_tensor_proto(instances, shape=instances.shape)
        )

        # Make prediction
        result = self.stub.Predict(request, timeout=10.0)
        return result

    def predict_batch(self, instances: List[np.ndarray]):
        """Batch prediction via gRPC"""
        batch = np.array(instances)
        return self.predict(batch)

    def close(self):
        """Close gRPC channel"""
        self.channel.close()

def test_grpc_client():
    """Test gRPC client"""
    client = TFServingGRPCClient()

    print("1. Testing single prediction...")
    test_image = np.random.rand(1, 28, 28, 1).astype(np.float32)

    start_time = time.time()
    result = client.predict(test_image)
    duration = time.time() - start_time

    print(f"Result outputs: {list(result.outputs.keys())}")
    print(f"Time: {duration*1000:.2f}ms")

    print("\n2. Testing batch prediction...")
    batch_images = np.random.rand(100, 28, 28, 1).astype(np.float32)

    start_time = time.time()
    result = client.predict(batch_images)
    duration = time.time() - start_time

    print(f"Batch size: 100")
    print(f"Total time: {duration*1000:.2f}ms")
    print(f"Time per image: {duration*1000/100:.2f}ms")

    # Compare with sequential
    print("\n3. Comparing with sequential requests...")
    start_time = time.time()
    for i in range(10):
        result = client.predict(test_image)
    duration = time.time() - start_time

    print(f"10 sequential requests: {duration*1000:.2f}ms")
    print(f"Time per request: {duration*1000/10:.2f}ms")

    client.close()

if __name__ == "__main__":
    test_grpc_client()
```

### Step 7: Create Benchmark Script

Create `benchmark.py`:

```python
import numpy as np
import time
from concurrent.futures import ThreadPoolExecutor
from rest_client import TFServingRESTClient
from grpc_client import TFServingGRPCClient
import statistics

def benchmark_rest(client, n_requests=100, batch_size=1):
    """Benchmark REST API"""
    test_image = np.random.rand(batch_size, 28, 28, 1).astype(np.float32)

    latencies = []
    for _ in range(n_requests):
        start = time.time()
        client.predict(test_image)
        latencies.append((time.time() - start) * 1000)

    return latencies

def benchmark_grpc(client, n_requests=100, batch_size=1):
    """Benchmark gRPC API"""
    test_image = np.random.rand(batch_size, 28, 28, 1).astype(np.float32)

    latencies = []
    for _ in range(n_requests):
        start = time.time()
        client.predict(test_image)
        latencies.append((time.time() - start) * 1000)

    return latencies

def benchmark_concurrent(client_type='rest', n_requests=100, n_workers=10):
    """Benchmark with concurrent requests"""
    def make_request(_):
        if client_type == 'rest':
            client = TFServingRESTClient()
            test_image = np.random.rand(1, 28, 28, 1).astype(np.float32)
            start = time.time()
            client.predict(test_image)
            return (time.time() - start) * 1000
        else:
            client = TFServingGRPCClient()
            test_image = np.random.rand(1, 28, 28, 1).astype(np.float32)
            start = time.time()
            client.predict(test_image)
            client.close()
            return (time.time() - start) * 1000

    with ThreadPoolExecutor(max_workers=n_workers) as executor:
        latencies = list(executor.map(make_request, range(n_requests)))

    return latencies

def print_stats(latencies, title):
    """Print benchmark statistics"""
    print(f"\n{title}")
    print("=" * 60)
    print(f"Requests: {len(latencies)}")
    print(f"Mean: {statistics.mean(latencies):.2f}ms")
    print(f"Median: {statistics.median(latencies):.2f}ms")
    print(f"Min: {min(latencies):.2f}ms")
    print(f"Max: {max(latencies):.2f}ms")
    print(f"Std Dev: {statistics.stdev(latencies):.2f}ms")
    print(f"p50: {np.percentile(latencies, 50):.2f}ms")
    print(f"p95: {np.percentile(latencies, 95):.2f}ms")
    print(f"p99: {np.percentile(latencies, 99):.2f}ms")
    print(f"Throughput: {1000 / statistics.mean(latencies):.2f} req/s")

def main():
    """Run comprehensive benchmarks"""
    print("TensorFlow Serving Benchmark")
    print("=" * 60)

    # REST API benchmarks
    print("\n1. REST API - Single Request")
    rest_client = TFServingRESTClient()
    rest_latencies = benchmark_rest(rest_client, n_requests=100, batch_size=1)
    print_stats(rest_latencies, "REST API (Single)")

    print("\n2. REST API - Batch (10)")
    rest_batch_latencies = benchmark_rest(rest_client, n_requests=100, batch_size=10)
    print_stats(rest_batch_latencies, "REST API (Batch=10)")

    # gRPC API benchmarks
    print("\n3. gRPC API - Single Request")
    grpc_client = TFServingGRPCClient()
    grpc_latencies = benchmark_grpc(grpc_client, n_requests=100, batch_size=1)
    print_stats(grpc_latencies, "gRPC API (Single)")
    grpc_client.close()

    print("\n4. gRPC API - Batch (10)")
    grpc_client = TFServingGRPCClient()
    grpc_batch_latencies = benchmark_grpc(grpc_client, n_requests=100, batch_size=10)
    print_stats(grpc_batch_latencies, "gRPC API (Batch=10)")
    grpc_client.close()

    # Concurrent requests
    print("\n5. REST API - Concurrent (10 workers)")
    rest_concurrent = benchmark_concurrent('rest', n_requests=100, n_workers=10)
    print_stats(rest_concurrent, "REST API (Concurrent)")

    print("\n6. gRPC API - Concurrent (10 workers)")
    grpc_concurrent = benchmark_concurrent('grpc', n_requests=100, n_workers=10)
    print_stats(grpc_concurrent, "gRPC API (Concurrent)")

if __name__ == "__main__":
    main()
```

### Step 8: Create Model Version Manager

Create `version_manager.py`:

```python
import os
import shutil
from train_and_export import train_model, export_model, save_metadata

def deploy_new_version():
    """Train and deploy a new model version"""

    # Get existing versions
    model_dir = 'models/mnist_model'
    versions = [int(d) for d in os.listdir(model_dir) if d.isdigit()]
    new_version = max(versions) + 1 if versions else 1

    print(f"Deploying new version: {new_version}")

    # Train model
    model, accuracy = train_model()

    # Export new version
    export_model(model, version=new_version)
    save_metadata(new_version, accuracy)

    print(f"\nNew version {new_version} deployed!")
    print("TensorFlow Serving will automatically load it.")

def rollback_version(target_version):
    """Rollback to a specific version"""
    model_dir = 'models/mnist_model'

    if not os.path.exists(f'{model_dir}/{target_version}'):
        print(f"Error: Version {target_version} does not exist")
        return

    print(f"Rolling back to version {target_version}")
    print("Remove newer versions or update model_config.config")

if __name__ == "__main__":
    import sys

    if len(sys.argv) > 1 and sys.argv[1] == 'deploy':
        deploy_new_version()
    elif len(sys.argv) > 2 and sys.argv[1] == 'rollback':
        rollback_version(int(sys.argv[2]))
    else:
        print("Usage:")
        print("  python version_manager.py deploy")
        print("  python version_manager.py rollback <version>")
```

### Step 9: Create Requirements

Create `requirements.txt`:

```txt
tensorflow==2.15.0
tensorflow-serving-api==2.15.0
grpcio==1.60.0
numpy==1.24.3
requests==2.31.0
```

### Step 10: Create Startup Script

Create `start_serving.sh`:

```bash
#!/bin/bash

echo "Starting TensorFlow Serving..."

# Pull latest TensorFlow Serving image
docker pull tensorflow/serving:latest

# Start with Docker Compose
docker-compose up -d

# Wait for service to be ready
echo "Waiting for TensorFlow Serving to start..."
sleep 10

# Check health
echo "Checking health..."
curl http://localhost:8501/v1/models/mnist_model

echo "\nTensorFlow Serving is ready!"
echo "REST API: http://localhost:8501"
echo "gRPC API: localhost:8500"
```

## Expected Outputs

### Model Status Response
```json
{
  "model_version_status": [
    {
      "version": "1",
      "state": "AVAILABLE",
      "status": {
        "error_code": "OK",
        "error_message": ""
      }
    }
  ]
}
```

### Prediction Response
```json
{
  "predictions": [
    [0.01, 0.02, 0.85, 0.03, 0.02, 0.01, 0.02, 0.02, 0.01, 0.01]
  ]
}
```

### Performance Metrics
```
REST API (Single): ~10-20ms
REST API (Batch=10): ~15-30ms (1.5-3ms per image)
gRPC API (Single): ~5-10ms
gRPC API (Batch=10): ~8-15ms (0.8-1.5ms per image)
Throughput: 500-2000 req/s
```

## Bonus Challenges

1. **Multi-Model Serving**: Serve multiple models simultaneously
2. **Model Warmup**: Implement model warmup requests
3. **GPU Support**: Configure GPU acceleration
4. **Monitoring**: Add Prometheus metrics exporter
5. **Kubernetes**: Deploy on Kubernetes with HPA
6. **Model Optimization**: Use TensorRT or TFLite
7. **A/B Testing**: Route traffic between versions
8. **Canary Deployment**: Gradual rollout of new versions
9. **Model Registry**: Integrate with MLflow
10. **Custom Ops**: Add custom TensorFlow operations

## Resources

- [TensorFlow Serving Documentation](https://www.tensorflow.org/tfx/guide/serving)
- [TensorFlow SavedModel Guide](https://www.tensorflow.org/guide/saved_model)
- [TF Serving REST API](https://www.tensorflow.org/tfx/serving/api_rest)
- [TF Serving gRPC API](https://www.tensorflow.org/tfx/serving/api_overview)

## Success Criteria

- [ ] Model exports in SavedModel format
- [ ] TensorFlow Serving starts successfully
- [ ] REST API accessible on port 8501
- [ ] gRPC API accessible on port 8500
- [ ] Model metadata endpoint works
- [ ] Single predictions complete < 20ms
- [ ] Batch predictions optimize throughput
- [ ] Multiple versions served simultaneously
- [ ] Hot-swapping works without downtime
- [ ] Batching configuration improves performance
- [ ] Both clients (REST/gRPC) work correctly
- [ ] Health checks pass

## Project Structure

```
tensorflow-serving/
├── models/
│   └── mnist_model/
│       ├── 1/
│       │   ├── saved_model.pb
│       │   ├── variables/
│       │   └── metadata.json
│       └── 2/ (optional)
├── train_and_export.py
├── rest_client.py
├── grpc_client.py
├── benchmark.py
├── version_manager.py
├── docker-compose.yml
├── model_config.config
├── batching_config.txt
├── start_serving.sh
├── requirements.txt
└── README.md
```
