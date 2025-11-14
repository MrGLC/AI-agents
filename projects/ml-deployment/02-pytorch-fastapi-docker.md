# Project 02: Deploy PyTorch Model with FastAPI and Docker

## Overview

Deploy a PyTorch deep learning model using FastAPI and containerize it with Docker. This project covers modern API development with async capabilities, Docker containerization, and best practices for deploying neural networks in production.

## Learning Objectives

- Train and export PyTorch models for inference
- Build async REST APIs with FastAPI
- Create production-ready Dockerfiles for ML applications
- Implement request validation with Pydantic
- Handle image/tensor inputs in API endpoints
- Optimize Docker images for ML workloads
- Implement health checks and monitoring in containers
- Use Docker Compose for multi-container setups

## Difficulty Level

**Intermediate** - Requires understanding of deep learning, async programming, and containerization.

## Technical Stack

- **ML Framework**: PyTorch 2.0+, torchvision
- **Web Framework**: FastAPI 0.104+
- **Server**: Uvicorn (ASGI server)
- **Validation**: Pydantic v2
- **Containerization**: Docker, Docker Compose
- **Testing**: pytest, httpx
- **Monitoring**: Prometheus metrics

## Requirements

### Model Requirements
- Train a PyTorch image classification model (e.g., ResNet, MobileNet)
- Export model for inference (TorchScript or state dict)
- Handle image preprocessing in the API

### API Requirements
- POST `/predict` - Image classification
- POST `/predict/batch` - Batch image processing
- GET `/health` - Health and readiness checks
- GET `/metrics` - Prometheus metrics
- GET `/docs` - Auto-generated API docs (Swagger)

### Docker Requirements
- Multi-stage build for smaller images
- Optimize layer caching
- Non-root user for security
- Health check configuration
- Environment-based configuration

## Step-by-Step Implementation

### Step 1: Train PyTorch Model

Create `train_model.py`:

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms, models
from torch.utils.data import DataLoader
import json
from datetime import datetime
import os

class ImageClassifier:
    def __init__(self, num_classes=10):
        # Use pretrained MobileNetV2 for efficiency
        self.model = models.mobilenet_v2(pretrained=True)

        # Modify final layer for our classes
        self.model.classifier[1] = nn.Linear(
            self.model.classifier[1].in_features,
            num_classes
        )

    def train(self, train_loader, val_loader, epochs=5):
        device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model = self.model.to(device)

        criterion = nn.CrossEntropyLoss()
        optimizer = optim.Adam(self.model.parameters(), lr=0.001)

        best_acc = 0.0

        for epoch in range(epochs):
            # Training phase
            self.model.train()
            train_loss = 0.0
            train_correct = 0

            for inputs, labels in train_loader:
                inputs, labels = inputs.to(device), labels.to(device)

                optimizer.zero_grad()
                outputs = self.model(inputs)
                loss = criterion(outputs, labels)
                loss.backward()
                optimizer.step()

                train_loss += loss.item()
                _, preds = torch.max(outputs, 1)
                train_correct += (preds == labels).sum().item()

            # Validation phase
            self.model.eval()
            val_correct = 0
            val_total = 0

            with torch.no_grad():
                for inputs, labels in val_loader:
                    inputs, labels = inputs.to(device), labels.to(device)
                    outputs = self.model(inputs)
                    _, preds = torch.max(outputs, 1)
                    val_correct += (preds == labels).sum().item()
                    val_total += labels.size(0)

            val_acc = val_correct / val_total
            train_acc = train_correct / len(train_loader.dataset)

            print(f'Epoch {epoch+1}/{epochs}:')
            print(f'  Train Loss: {train_loss/len(train_loader):.4f}, Acc: {train_acc:.4f}')
            print(f'  Val Acc: {val_acc:.4f}')

            if val_acc > best_acc:
                best_acc = val_acc

        return best_acc

def prepare_data():
    """Prepare CIFAR-10 dataset"""
    transform = transforms.Compose([
        transforms.Resize(224),  # MobileNet expects 224x224
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406],
                           std=[0.229, 0.224, 0.225])
    ])

    train_dataset = datasets.CIFAR10(
        root='./data',
        train=True,
        download=True,
        transform=transform
    )

    val_dataset = datasets.CIFAR10(
        root='./data',
        train=False,
        download=True,
        transform=transform
    )

    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=32, shuffle=False)

    return train_loader, val_loader, train_dataset.classes

def main():
    print("Preparing data...")
    train_loader, val_loader, class_names = prepare_data()

    print("Training model...")
    classifier = ImageClassifier(num_classes=10)
    accuracy = classifier.train(train_loader, val_loader, epochs=5)

    # Save model
    os.makedirs('models', exist_ok=True)

    # Save as TorchScript (recommended for production)
    classifier.model.eval()
    example_input = torch.rand(1, 3, 224, 224)
    traced_model = torch.jit.trace(classifier.model, example_input)
    torch.jit.save(traced_model, 'models/cifar10_mobilenet.pt')

    # Also save state dict as backup
    torch.save(classifier.model.state_dict(), 'models/cifar10_mobilenet_state.pth')

    # Save metadata
    metadata = {
        'model_type': 'MobileNetV2',
        'num_classes': 10,
        'class_names': class_names,
        'input_size': [224, 224],
        'accuracy': float(accuracy),
        'trained_at': datetime.now().isoformat(),
        'pytorch_version': torch.__version__,
        'preprocessing': {
            'mean': [0.485, 0.456, 0.406],
            'std': [0.229, 0.224, 0.225]
        }
    }

    with open('models/metadata.json', 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"\nModel saved! Accuracy: {accuracy:.4f}")
    print(f"TorchScript: models/cifar10_mobilenet.pt")
    print(f"State Dict: models/cifar10_mobilenet_state.pth")

if __name__ == "__main__":
    main()
```

### Step 2: Create FastAPI Application

Create `app.py`:

```python
from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import List, Optional
import torch
from torchvision import transforms
from PIL import Image
import io
import json
import logging
from datetime import datetime
import asyncio
from prometheus_client import Counter, Histogram, generate_latest
from fastapi.responses import Response

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Prometheus metrics
PREDICTIONS = Counter('predictions_total', 'Total predictions made')
PREDICTION_TIME = Histogram('prediction_duration_seconds', 'Prediction duration')
ERRORS = Counter('prediction_errors_total', 'Total prediction errors')

# Initialize FastAPI
app = FastAPI(
    title="PyTorch Image Classification API",
    description="Deploy MobileNetV2 for CIFAR-10 classification",
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

# Global model and metadata
model = None
metadata = None
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
transform = None

# Pydantic models
class PredictionResponse(BaseModel):
    class_id: int
    class_name: str
    confidence: float
    probabilities: dict
    timestamp: str

class BatchPredictionResponse(BaseModel):
    predictions: List[PredictionResponse]
    count: int
    timestamp: str

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool
    device: str
    timestamp: str

class ModelInfo(BaseModel):
    model_type: str
    num_classes: int
    class_names: List[str]
    accuracy: float
    pytorch_version: str

@app.on_event("startup")
async def load_model():
    """Load model on startup"""
    global model, metadata, transform

    try:
        # Load metadata
        with open('models/metadata.json', 'r') as f:
            metadata = json.load(f)

        # Load TorchScript model
        model = torch.jit.load('models/cifar10_mobilenet.pt')
        model = model.to(device)
        model.eval()

        # Setup preprocessing
        mean = metadata['preprocessing']['mean']
        std = metadata['preprocessing']['std']
        transform = transforms.Compose([
            transforms.Resize(tuple(metadata['input_size'])),
            transforms.ToTensor(),
            transforms.Normalize(mean=mean, std=std)
        ])

        logger.info(f"Model loaded successfully on {device}")
        logger.info(f"Classes: {metadata['class_names']}")

    except Exception as e:
        logger.error(f"Failed to load model: {e}")
        raise

def preprocess_image(image_bytes: bytes) -> torch.Tensor:
    """Preprocess image for model"""
    try:
        image = Image.open(io.BytesIO(image_bytes)).convert('RGB')
        tensor = transform(image).unsqueeze(0)
        return tensor.to(device)
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Invalid image: {str(e)}")

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    return HealthResponse(
        status="healthy" if model is not None else "unhealthy",
        model_loaded=model is not None,
        device=str(device),
        timestamp=datetime.now().isoformat()
    )

@app.get("/model-info", response_model=ModelInfo)
async def get_model_info():
    """Get model information"""
    if metadata is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    return ModelInfo(
        model_type=metadata['model_type'],
        num_classes=metadata['num_classes'],
        class_names=metadata['class_names'],
        accuracy=metadata['accuracy'],
        pytorch_version=metadata['pytorch_version']
    )

@app.post("/predict", response_model=PredictionResponse)
async def predict(file: UploadFile = File(...)):
    """Single image prediction"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    try:
        # Read and preprocess image
        image_bytes = await file.read()
        input_tensor = preprocess_image(image_bytes)

        # Make prediction
        with PREDICTION_TIME.time():
            with torch.no_grad():
                output = model(input_tensor)
                probabilities = torch.nn.functional.softmax(output, dim=1)
                confidence, predicted_idx = torch.max(probabilities, 1)

        # Format response
        class_id = int(predicted_idx.item())
        class_name = metadata['class_names'][class_id]

        probs_dict = {
            metadata['class_names'][i]: float(prob)
            for i, prob in enumerate(probabilities[0])
        }

        PREDICTIONS.inc()

        return PredictionResponse(
            class_id=class_id,
            class_name=class_name,
            confidence=float(confidence.item()),
            probabilities=probs_dict,
            timestamp=datetime.now().isoformat()
        )

    except HTTPException:
        raise
    except Exception as e:
        ERRORS.inc()
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail="Prediction failed")

@app.post("/predict/batch", response_model=BatchPredictionResponse)
async def batch_predict(files: List[UploadFile] = File(...)):
    """Batch image prediction"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")

    try:
        predictions = []

        # Process images in parallel
        image_bytes_list = await asyncio.gather(*[file.read() for file in files])

        for image_bytes in image_bytes_list:
            try:
                input_tensor = preprocess_image(image_bytes)

                with torch.no_grad():
                    output = model(input_tensor)
                    probabilities = torch.nn.functional.softmax(output, dim=1)
                    confidence, predicted_idx = torch.max(probabilities, 1)

                class_id = int(predicted_idx.item())
                class_name = metadata['class_names'][class_id]

                probs_dict = {
                    metadata['class_names'][i]: float(prob)
                    for i, prob in enumerate(probabilities[0])
                }

                predictions.append(PredictionResponse(
                    class_id=class_id,
                    class_name=class_name,
                    confidence=float(confidence.item()),
                    probabilities=probs_dict,
                    timestamp=datetime.now().isoformat()
                ))

                PREDICTIONS.inc()

            except Exception as e:
                logger.error(f"Failed to process image: {e}")
                ERRORS.inc()

        return BatchPredictionResponse(
            predictions=predictions,
            count=len(predictions),
            timestamp=datetime.now().isoformat()
        )

    except Exception as e:
        logger.error(f"Batch prediction error: {e}")
        raise HTTPException(status_code=500, detail="Batch prediction failed")

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return Response(content=generate_latest(), media_type="text/plain")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 3: Create Dockerfile

Create `Dockerfile`:

```dockerfile
# Multi-stage build for smaller final image

# Stage 1: Build stage
FROM python:3.11-slim as builder

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime stage
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Copy Python dependencies from builder
COPY --from=builder /root/.local /root/.local

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

# Copy application code
COPY app.py .
COPY models/ models/

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

# Run application
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### Step 4: Create Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  api:
    build: .
    container_name: pytorch-api
    ports:
      - "8000:8000"
    environment:
      - WORKERS=4
      - LOG_LEVEL=info
    volumes:
      - ./models:/app/models:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Optional: Prometheus for metrics
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

volumes:
  prometheus_data:
```

### Step 5: Create Requirements

Create `requirements.txt`:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6
torch==2.1.0
torchvision==0.16.0
Pillow==10.1.0
pydantic==2.5.0
prometheus-client==0.19.0
httpx==0.25.2
pytest==7.4.3
```

### Step 6: Create Prometheus Config

Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'pytorch-api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: '/metrics'
```

### Step 7: Create Test Suite

Create `test_api.py`:

```python
import pytest
from httpx import AsyncClient
from app import app
import io
from PIL import Image

@pytest.fixture
def sample_image():
    """Create sample test image"""
    img = Image.new('RGB', (224, 224), color='red')
    img_bytes = io.BytesIO()
    img.save(img_bytes, format='PNG')
    img_bytes.seek(0)
    return img_bytes

@pytest.mark.asyncio
async def test_health_check():
    """Test health endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")
        assert response.status_code == 200
        data = response.json()
        assert data["model_loaded"] == True

@pytest.mark.asyncio
async def test_model_info():
    """Test model info endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/model-info")
        assert response.status_code == 200
        data = response.json()
        assert "class_names" in data
        assert len(data["class_names"]) == 10

@pytest.mark.asyncio
async def test_predict(sample_image):
    """Test prediction endpoint"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        files = {"file": ("test.png", sample_image, "image/png")}
        response = await client.post("/predict", files=files)
        assert response.status_code == 200
        data = response.json()
        assert "class_name" in data
        assert "confidence" in data
        assert "probabilities" in data

@pytest.mark.asyncio
async def test_batch_predict(sample_image):
    """Test batch prediction"""
    async with AsyncClient(app=app, base_url="http://test") as client:
        files = [
            ("files", ("test1.png", sample_image, "image/png")),
            ("files", ("test2.png", sample_image, "image/png"))
        ]
        response = await client.post("/predict/batch", files=files)
        assert response.status_code == 200
        data = response.json()
        assert "predictions" in data
        assert data["count"] >= 1
```

### Step 8: Create Client Example

Create `client_example.py`:

```python
import requests
from pathlib import Path

BASE_URL = "http://localhost:8000"

def test_api():
    """Test PyTorch API"""

    # Health check
    print("Testing health check...")
    response = requests.get(f"{BASE_URL}/health")
    print(f"Health: {response.json()}\n")

    # Model info
    print("Testing model info...")
    response = requests.get(f"{BASE_URL}/model-info")
    print(f"Model Info: {response.json()}\n")

    # Single prediction
    print("Testing single prediction...")
    # Use a sample image
    image_path = "test_images/sample.jpg"
    if Path(image_path).exists():
        with open(image_path, 'rb') as f:
            files = {'file': f}
            response = requests.post(f"{BASE_URL}/predict", files=files)
            print(f"Prediction: {response.json()}\n")

    # Metrics
    print("Testing metrics...")
    response = requests.get(f"{BASE_URL}/metrics")
    print(f"Metrics (first 500 chars):\n{response.text[:500]}\n")

if __name__ == "__main__":
    test_api()
```

### Step 9: Build and Run

Create `Makefile`:

```makefile
.PHONY: train build run test clean

train:
	python train_model.py

build:
	docker-compose build

run:
	docker-compose up -d

logs:
	docker-compose logs -f api

test:
	pytest test_api.py -v

stop:
	docker-compose down

clean:
	docker-compose down -v
	rm -rf models/ data/

shell:
	docker exec -it pytorch-api /bin/bash

metrics:
	curl http://localhost:8000/metrics
```

## Expected Outputs

### Prediction Response
```json
{
  "class_id": 3,
  "class_name": "cat",
  "confidence": 0.89,
  "probabilities": {
    "airplane": 0.01,
    "automobile": 0.02,
    "bird": 0.05,
    "cat": 0.89,
    "deer": 0.01,
    "dog": 0.01,
    "frog": 0.00,
    "horse": 0.01,
    "ship": 0.00,
    "truck": 0.00
  },
  "timestamp": "2024-11-14T10:30:00"
}
```

### Docker Build Output
```
Successfully built pytorch-api
Image size: ~2.5GB (optimized with multi-stage build)
```

## Bonus Challenges

1. **Model Quantization**: Implement INT8 quantization to reduce model size
2. **GPU Support**: Add NVIDIA runtime support for GPU inference
3. **Model Caching**: Implement LRU cache for frequently used models
4. **Horizontal Scaling**: Deploy multiple containers with NGINX load balancer
5. **Streaming Inference**: Add WebSocket endpoint for real-time video processing
6. **Model Warmup**: Implement model warmup on container startup
7. **TensorRT**: Convert model to TensorRT for faster inference
8. **Kubernetes**: Create K8s manifests for deployment
9. **CI/CD**: Add GitHub Actions for automated Docker builds
10. **Monitoring Dashboard**: Create Grafana dashboard for metrics

## Resources

- [PyTorch Production Deployment](https://pytorch.org/tutorials/intermediate/flask_rest_api_tutorial.html)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [TorchScript Guide](https://pytorch.org/docs/stable/jit.html)

## Success Criteria

- [ ] Model trains and exports to TorchScript
- [ ] FastAPI app starts and serves predictions
- [ ] Docker image builds successfully (< 3GB)
- [ ] All API endpoints return correct responses
- [ ] Swagger UI accessible at /docs
- [ ] Health checks pass in Docker
- [ ] Tests pass with pytest
- [ ] Prometheus metrics exposed
- [ ] Container runs as non-root user
- [ ] Multi-stage build optimizes image size
- [ ] Async endpoints handle concurrent requests
- [ ] API handles image uploads correctly

## Project Structure

```
pytorch-fastapi-docker/
├── models/
│   ├── cifar10_mobilenet.pt
│   ├── cifar10_mobilenet_state.pth
│   └── metadata.json
├── data/
├── app.py
├── train_model.py
├── test_api.py
├── client_example.py
├── Dockerfile
├── docker-compose.yml
├── prometheus.yml
├── requirements.txt
├── Makefile
└── README.md
```
