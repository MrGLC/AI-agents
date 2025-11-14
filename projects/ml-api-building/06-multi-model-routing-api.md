# Project 6: Build Multi-Model API with Routing

## Overview
Create an API that serves multiple ML models simultaneously with intelligent routing based on model type, version, or capabilities. This architecture allows you to serve different models for different tasks (classification, detection, segmentation) through a unified interface.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Design multi-model serving architecture
- Implement model registry and versioning
- Create routing logic based on model capabilities
- Manage multiple models in memory efficiently
- Implement model loading strategies (lazy/eager)
- Handle model-specific preprocessing pipelines
- Design unified response format across models

## Technical Stack
- **Framework**: FastAPI
- **ML Frameworks**: PyTorch, TensorFlow (multi-framework support)
- **Model Management**: Custom registry
- **Caching**: functools.lru_cache
- **Configuration**: YAML or JSON
- **Testing**: pytest

## Project Requirements

### 1. Model Registry
- Dynamic model loading from configuration
- Model metadata (type, version, capabilities)
- Model versioning (v1, v2, latest)
- Model health checks
- Hot-swapping models without restart

### 2. API Endpoints
- `GET /models` - List all available models
- `GET /models/{model_id}` - Get model details
- `POST /predict/{model_id}` - Predict with specific model
- `POST /predict/auto` - Auto-route to best model
- `POST /models/reload` - Reload models from config
- `GET /models/{model_id}/health` - Model health check

### 3. Routing Strategies
- By model ID (explicit selection)
- By model type (classification, detection, etc.)
- By model version (v1, v2, latest)
- Auto-routing based on input characteristics
- A/B testing support

### 4. Model Types Supported
- Image Classification
- Object Detection
- Image Segmentation
- Text Classification
- Multi-modal models

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install fastapi uvicorn pydantic
pip install torch torchvision
pip install pyyaml pillow
pip install pytest httpx
```

### Step 2: Project Structure
```bash
mkdir multi_model_api
cd multi_model_api

touch main.py model_registry.py model_loader.py
touch models.py config.py utils.py
touch models_config.yaml test_multi_model.py
mkdir -p models/weights
```

### Step 3: Models Configuration (models_config.yaml)
```yaml
models:
  - id: resnet50-v1
    name: ResNet50 Classifier
    type: classification
    version: "1.0"
    framework: pytorch
    model_class: torchvision.models.resnet50
    pretrained: true
    input_size: [224, 224]
    num_classes: 1000
    enabled: true
    load_on_startup: true

  - id: resnet18-v1
    name: ResNet18 Classifier (Fast)
    type: classification
    version: "1.0"
    framework: pytorch
    model_class: torchvision.models.resnet18
    pretrained: true
    input_size: [224, 224]
    num_classes: 1000
    enabled: true
    load_on_startup: false  # Lazy loading

  - id: mobilenet-v2
    name: MobileNetV2 (Mobile-optimized)
    type: classification
    version: "2.0"
    framework: pytorch
    model_class: torchvision.models.mobilenet_v2
    pretrained: true
    input_size: [224, 224]
    num_classes: 1000
    enabled: true
    load_on_startup: false
    tags:
      - mobile
      - fast
      - lightweight

routing_rules:
  default_model: resnet50-v1
  auto_routing:
    classification: resnet50-v1
  version_aliases:
    latest: resnet50-v1
    fast: resnet18-v1
    mobile: mobilenet-v2
```

### Step 4: Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field
from typing import List, Dict, Optional, Literal
from datetime import datetime
from enum import Enum

class ModelType(str, Enum):
    CLASSIFICATION = "classification"
    DETECTION = "detection"
    SEGMENTATION = "segmentation"
    TEXT = "text"

class ModelStatus(str, Enum):
    LOADED = "loaded"
    UNLOADED = "unloaded"
    LOADING = "loading"
    ERROR = "error"

class ModelInfo(BaseModel):
    """Model information"""
    id: str
    name: str
    type: ModelType
    version: str
    framework: str
    input_size: List[int]
    num_classes: Optional[int] = None
    enabled: bool = True
    status: ModelStatus
    tags: List[str] = []
    loaded_at: Optional[datetime] = None
    memory_usage_mb: Optional[float] = None

class ModelListResponse(BaseModel):
    """List of models"""
    total: int
    models: List[ModelInfo]

class PredictionRequest(BaseModel):
    """Prediction request"""
    image: str  # base64
    top_k: int = Field(default=5, ge=1, le=20)
    threshold: Optional[float] = Field(default=None, ge=0.0, le=1.0)

class AutoRouteRequest(PredictionRequest):
    """Auto-routing prediction request"""
    task_type: Optional[ModelType] = None
    prefer_fast: bool = False
    prefer_accurate: bool = False

class Prediction(BaseModel):
    """Single prediction result"""
    class_name: str
    confidence: float
    rank: int

class PredictionResponse(BaseModel):
    """Prediction response"""
    model_id: str
    model_name: str
    predictions: List[Prediction]
    top_prediction: str
    confidence: float
    processing_time: float
    timestamp: datetime

class ModelHealthResponse(BaseModel):
    """Model health check response"""
    model_id: str
    status: ModelStatus
    healthy: bool
    last_prediction: Optional[datetime] = None
    total_predictions: int
    avg_latency_ms: Optional[float] = None
    error: Optional[str] = None
```

### Step 5: Model Loader (model_loader.py)
```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from typing import Dict, Any, Optional
import importlib
from PIL import Image
import io
import base64
import time

class ModelLoader:
    """Handles loading and managing ML models"""

    def __init__(self):
        self.loaded_models: Dict[str, Any] = {}
        self.model_configs: Dict[str, dict] = {}
        self.transforms: Dict[str, transforms.Compose] = {}

    def load_model(self, config: dict) -> Any:
        """Load a model from configuration"""
        model_id = config['id']
        framework = config.get('framework', 'pytorch')

        if framework == 'pytorch':
            return self._load_pytorch_model(config)
        else:
            raise ValueError(f"Unsupported framework: {framework}")

    def _load_pytorch_model(self, config: dict) -> Any:
        """Load PyTorch model"""
        model_class = config['model_class']

        # Dynamically import model class
        if model_class.startswith('torchvision.models'):
            model_name = model_class.split('.')[-1]
            model_fn = getattr(models, model_name)
        else:
            raise ValueError(f"Unsupported model class: {model_class}")

        # Load model
        print(f"Loading {config['name']}...")
        model = model_fn(pretrained=config.get('pretrained', True))
        model.eval()

        # Move to GPU if available
        device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        model.to(device)

        # Store configuration
        self.model_configs[config['id']] = config

        # Create transform
        input_size = config.get('input_size', [224, 224])
        self.transforms[config['id']] = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(input_size[0]),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])

        print(f"Model {config['name']} loaded successfully on {device}")
        return model

    def predict(
        self,
        model_id: str,
        image_base64: str,
        top_k: int = 5
    ) -> dict:
        """Make prediction with specified model"""
        if model_id not in self.loaded_models:
            raise ValueError(f"Model {model_id} not loaded")

        model = self.loaded_models[model_id]
        transform = self.transforms[model_id]

        # Decode and preprocess image
        image_data = base64.b64decode(image_base64)
        image = Image.open(io.BytesIO(image_data)).convert('RGB')
        img_tensor = transform(image).unsqueeze(0)

        # Move to same device as model
        device = next(model.parameters()).device
        img_tensor = img_tensor.to(device)

        # Predict
        start_time = time.time()
        with torch.no_grad():
            outputs = model(img_tensor)
            probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

        processing_time = time.time() - start_time

        # Get top K predictions
        top_probs, top_indices = torch.topk(probabilities, top_k)

        # Dummy class names
        classes = [f"class_{i}" for i in range(1000)]

        predictions = [
            {
                'class_name': classes[idx],
                'confidence': float(prob),
                'rank': i + 1
            }
            for i, (prob, idx) in enumerate(zip(top_probs, top_indices))
        ]

        return {
            'predictions': predictions,
            'top_prediction': predictions[0]['class_name'],
            'confidence': predictions[0]['confidence'],
            'processing_time': processing_time
        }

    def get_model_memory_usage(self, model_id: str) -> Optional[float]:
        """Get model memory usage in MB"""
        if model_id not in self.loaded_models:
            return None

        model = self.loaded_models[model_id]
        param_size = sum(p.numel() * p.element_size() for p in model.parameters())
        buffer_size = sum(b.numel() * b.element_size() for b in model.buffers())
        size_mb = (param_size + buffer_size) / 1024 / 1024

        return size_mb

# Global model loader
model_loader = ModelLoader()
```

### Step 6: Model Registry (model_registry.py)
```python
import yaml
from typing import Dict, List, Optional
from datetime import datetime
from pathlib import Path

from models import ModelInfo, ModelStatus, ModelType
from model_loader import model_loader

class ModelRegistry:
    """Central registry for managing multiple models"""

    def __init__(self, config_path: str = "models_config.yaml"):
        self.config_path = config_path
        self.models: Dict[str, ModelInfo] = {}
        self.routing_rules: dict = {}
        self.prediction_counts: Dict[str, int] = {}
        self.last_predictions: Dict[str, datetime] = {}

    def load_config(self):
        """Load models configuration"""
        with open(self.config_path, 'r') as f:
            config = yaml.safe_load(f)

        self.routing_rules = config.get('routing_rules', {})

        # Register all models
        for model_config in config.get('models', []):
            model_info = ModelInfo(
                id=model_config['id'],
                name=model_config['name'],
                type=ModelType(model_config['type']),
                version=model_config['version'],
                framework=model_config['framework'],
                input_size=model_config['input_size'],
                num_classes=model_config.get('num_classes'),
                enabled=model_config.get('enabled', True),
                status=ModelStatus.UNLOADED,
                tags=model_config.get('tags', [])
            )
            self.models[model_config['id']] = model_info

            # Load model if configured
            if model_config.get('load_on_startup', False):
                self.load_model(model_config['id'], model_config)

            self.prediction_counts[model_config['id']] = 0

    def load_model(self, model_id: str, config: dict = None):
        """Load a specific model"""
        if model_id not in self.models:
            raise ValueError(f"Model {model_id} not found in registry")

        self.models[model_id].status = ModelStatus.LOADING

        try:
            # Load model
            if config is None:
                raise ValueError("Model config required for loading")

            model = model_loader.load_model(config)
            model_loader.loaded_models[model_id] = model

            # Update status
            self.models[model_id].status = ModelStatus.LOADED
            self.models[model_id].loaded_at = datetime.utcnow()
            self.models[model_id].memory_usage_mb = model_loader.get_model_memory_usage(model_id)

        except Exception as e:
            self.models[model_id].status = ModelStatus.ERROR
            raise e

    def get_model(self, model_id: str) -> Optional[ModelInfo]:
        """Get model info"""
        return self.models.get(model_id)

    def list_models(
        self,
        model_type: Optional[ModelType] = None,
        enabled_only: bool = True
    ) -> List[ModelInfo]:
        """List all models with optional filtering"""
        models = list(self.models.values())

        if model_type:
            models = [m for m in models if m.type == model_type]

        if enabled_only:
            models = [m for m in models if m.enabled]

        return models

    def auto_route(
        self,
        task_type: Optional[ModelType] = None,
        prefer_fast: bool = False,
        prefer_accurate: bool = False
    ) -> str:
        """Auto-route to best model based on criteria"""
        if task_type:
            # Use routing rules
            model_id = self.routing_rules.get('auto_routing', {}).get(
                task_type.value,
                self.routing_rules.get('default_model')
            )
        else:
            # Use preferences
            if prefer_fast:
                model_id = self.routing_rules.get('version_aliases', {}).get('fast')
            elif prefer_accurate:
                model_id = self.routing_rules.get('default_model')
            else:
                model_id = self.routing_rules.get('default_model')

        # Ensure model is loaded
        if model_id not in model_loader.loaded_models:
            raise ValueError(f"Selected model {model_id} not loaded")

        return model_id

    def record_prediction(self, model_id: str):
        """Record prediction for metrics"""
        self.prediction_counts[model_id] = self.prediction_counts.get(model_id, 0) + 1
        self.last_predictions[model_id] = datetime.utcnow()

    def get_model_health(self, model_id: str) -> dict:
        """Get model health information"""
        if model_id not in self.models:
            raise ValueError(f"Model {model_id} not found")

        model_info = self.models[model_id]

        return {
            'model_id': model_id,
            'status': model_info.status,
            'healthy': model_info.status == ModelStatus.LOADED,
            'last_prediction': self.last_predictions.get(model_id),
            'total_predictions': self.prediction_counts.get(model_id, 0),
            'avg_latency_ms': None  # Could track this
        }

# Global registry
registry = ModelRegistry()
```

### Step 7: Main Application (main.py)
```python
from fastapi import FastAPI, HTTPException, File, UploadFile
from typing import Optional, List
from datetime import datetime
import base64

from models import (
    ModelListResponse, ModelInfo, PredictionRequest,
    PredictionResponse, Prediction, ModelHealthResponse,
    AutoRouteRequest, ModelType
)
from model_registry import registry
from model_loader import model_loader

app = FastAPI(
    title="Multi-Model ML API",
    version="1.0.0",
    description="Serve multiple ML models with intelligent routing"
)

@app.on_event("startup")
async def startup_event():
    """Load models on startup"""
    try:
        registry.load_config()
        print(f"Loaded {len(registry.models)} models")
    except Exception as e:
        print(f"Error loading models: {e}")
        raise

@app.get("/")
async def root():
    """API information"""
    return {
        "message": "Multi-Model ML API",
        "total_models": len(registry.models),
        "loaded_models": len(model_loader.loaded_models),
        "endpoints": {
            "list_models": "/models",
            "predict": "/predict/{model_id}",
            "auto_predict": "/predict/auto"
        }
    }

@app.get("/models", response_model=ModelListResponse)
async def list_models(
    type: Optional[ModelType] = None,
    enabled_only: bool = True
):
    """List all available models"""
    models = registry.list_models(model_type=type, enabled_only=enabled_only)

    return ModelListResponse(
        total=len(models),
        models=models
    )

@app.get("/models/{model_id}", response_model=ModelInfo)
async def get_model(model_id: str):
    """Get specific model information"""
    model = registry.get_model(model_id)

    if not model:
        raise HTTPException(
            status_code=404,
            detail=f"Model {model_id} not found"
        )

    return model

@app.post("/predict/{model_id}", response_model=PredictionResponse)
async def predict_with_model(
    model_id: str,
    request: PredictionRequest
):
    """
    Make prediction with specific model

    - **model_id**: ID of the model to use
    - **image**: Base64 encoded image
    - **top_k**: Number of top predictions to return
    """
    # Validate model exists and is loaded
    model = registry.get_model(model_id)

    if not model:
        raise HTTPException(
            status_code=404,
            detail=f"Model {model_id} not found"
        )

    if model.status != "loaded":
        raise HTTPException(
            status_code=400,
            detail=f"Model {model_id} is not loaded. Status: {model.status}"
        )

    try:
        # Make prediction
        result = model_loader.predict(
            model_id,
            request.image,
            top_k=request.top_k
        )

        # Record prediction
        registry.record_prediction(model_id)

        # Format response
        predictions = [
            Prediction(**pred)
            for pred in result['predictions']
        ]

        return PredictionResponse(
            model_id=model_id,
            model_name=model.name,
            predictions=predictions,
            top_prediction=result['top_prediction'],
            confidence=result['confidence'],
            processing_time=result['processing_time'],
            timestamp=datetime.utcnow()
        )

    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Error during prediction: {str(e)}"
        )

@app.post("/predict/auto", response_model=PredictionResponse)
async def predict_auto_route(request: AutoRouteRequest):
    """
    Auto-route to best model based on request

    - **task_type**: Type of ML task (optional)
    - **prefer_fast**: Prefer faster model
    - **prefer_accurate**: Prefer more accurate model
    """
    try:
        # Auto-route to best model
        model_id = registry.auto_route(
            task_type=request.task_type,
            prefer_fast=request.prefer_fast,
            prefer_accurate=request.prefer_accurate
        )

        # Make prediction
        model = registry.get_model(model_id)
        result = model_loader.predict(
            model_id,
            request.image,
            top_k=request.top_k
        )

        registry.record_prediction(model_id)

        predictions = [
            Prediction(**pred)
            for pred in result['predictions']
        ]

        return PredictionResponse(
            model_id=model_id,
            model_name=model.name,
            predictions=predictions,
            top_prediction=result['top_prediction'],
            confidence=result['confidence'],
            processing_time=result['processing_time'],
            timestamp=datetime.utcnow()
        )

    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Error during auto-routing: {str(e)}"
        )

@app.get("/models/{model_id}/health", response_model=ModelHealthResponse)
async def check_model_health(model_id: str):
    """Check health of specific model"""
    try:
        health = registry.get_model_health(model_id)
        return ModelHealthResponse(**health)
    except Exception as e:
        raise HTTPException(
            status_code=404,
            detail=str(e)
        )

@app.post("/models/reload")
async def reload_models():
    """Reload all models from configuration"""
    try:
        # Clear current models
        model_loader.loaded_models.clear()

        # Reload configuration
        registry.load_config()

        return {
            "message": "Models reloaded successfully",
            "total_models": len(registry.models),
            "loaded_models": len(model_loader.loaded_models)
        }
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Error reloading models: {str(e)}"
        )

@app.get("/health")
async def health_check():
    """Overall API health check"""
    return {
        "status": "healthy",
        "total_models": len(registry.models),
        "loaded_models": len(model_loader.loaded_models),
        "total_predictions": sum(registry.prediction_counts.values())
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 8: Run the Application
```bash
# Start the server
uvicorn main:app --reload --port 8000

# Test endpoints
curl http://localhost:8000/models
curl http://localhost:8000/models/resnet50-v1

# Make prediction with specific model
curl -X POST "http://localhost:8000/predict/resnet50-v1" \
  -H "Content-Type: application/json" \
  -d '{"image": "<base64-encoded-image>", "top_k": 5}'

# Auto-route prediction
curl -X POST "http://localhost:8000/predict/auto" \
  -H "Content-Type: application/json" \
  -d '{"image": "<base64-encoded-image>", "prefer_fast": true}'
```

## Expected Outputs

### 1. List Models Response
```json
{
  "total": 3,
  "models": [
    {
      "id": "resnet50-v1",
      "name": "ResNet50 Classifier",
      "type": "classification",
      "version": "1.0",
      "framework": "pytorch",
      "input_size": [224, 224],
      "num_classes": 1000,
      "enabled": true,
      "status": "loaded",
      "tags": [],
      "loaded_at": "2025-11-14T10:00:00",
      "memory_usage_mb": 97.5
    }
  ]
}
```

### 2. Prediction Response
```json
{
  "model_id": "resnet50-v1",
  "model_name": "ResNet50 Classifier",
  "predictions": [
    {
      "class_name": "golden_retriever",
      "confidence": 0.87,
      "rank": 1
    }
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "processing_time": 0.023,
  "timestamp": "2025-11-14T10:30:00"
}
```

## Bonus Challenges

- [ ] Add A/B testing framework
- [ ] Implement model ensemble predictions
- [ ] Add model performance comparison
- [ ] Create model benchmark suite
- [ ] Implement model warm-up on startup
- [ ] Add model caching with TTL
- [ ] Create model deployment pipeline
- [ ] Add model monitoring and alerting
- [ ] Implement canary deployments
- [ ] Add TensorFlow and ONNX support
- [ ] Create model optimization (quantization)
- [ ] Add distributed model serving

## Resources

- [FastAPI Advanced Features](https://fastapi.tiangolo.com/advanced/)
- [PyTorch Model Zoo](https://pytorch.org/serve/model_zoo.html)
- [Model Serving Best Practices](https://neptune.ai/blog/ml-model-serving-best-tools)
- [Multi-Model Serving](https://www.tensorflow.org/tfx/serving/serving_advanced)

## Success Criteria

- [ ] Multiple models load successfully
- [ ] Models can be queried and listed
- [ ] Predictions work with each model
- [ ] Auto-routing selects correct model
- [ ] Lazy loading works correctly
- [ ] Model health checks are accurate
- [ ] Memory usage is tracked
- [ ] Configuration reloading works
- [ ] All tests pass
- [ ] API documentation is complete
