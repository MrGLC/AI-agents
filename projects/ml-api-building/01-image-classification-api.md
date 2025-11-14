# Project 1: Build RESTful API for Image Classification

## Overview
Create a production-ready REST API that serves an image classification model using FastAPI. The API will accept image uploads, perform predictions, and return classification results with confidence scores.

## Difficulty Level
Beginner to Intermediate

## Learning Objectives
- Build a RESTful API using FastAPI
- Integrate a pre-trained ML model into an API
- Implement request/response validation with Pydantic
- Handle file uploads and image preprocessing
- Create health check and info endpoints
- Write API tests with pytest
- Generate automatic API documentation

## Technical Stack
- **Framework**: FastAPI
- **ML Framework**: TensorFlow/Keras or PyTorch
- **Validation**: Pydantic
- **Server**: Uvicorn
- **Testing**: pytest, httpx
- **Model**: Pre-trained image classifier (ResNet, MobileNet, etc.)

## Project Requirements

### 1. Core API Endpoints
- `POST /predict` - Accept image and return predictions
- `GET /health` - Health check endpoint
- `GET /info` - Model information (version, classes, etc.)
- `GET /` - API documentation redirect

### 2. Request Validation
- Accept multiple image formats (JPEG, PNG, etc.)
- Validate file size (max 10MB)
- Validate image dimensions
- Return clear error messages for invalid inputs

### 3. Response Format
- JSON response with predictions
- Include confidence scores
- Add metadata (timestamp, model version)
- Proper HTTP status codes

### 4. Testing
- Unit tests for preprocessing functions
- Integration tests for API endpoints
- Test error handling scenarios
- Test with various image formats

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn pillow torch torchvision pydantic python-multipart
pip install pytest httpx  # For testing
```

### Step 2: Create Project Structure
```bash
mkdir image_classifier_api
cd image_classifier_api
touch main.py models.py config.py utils.py test_api.py
mkdir -p models/weights
```

### Step 3: Define Pydantic Models (models.py)
```python
from pydantic import BaseModel, Field
from typing import List, Dict
from datetime import datetime

class PredictionResponse(BaseModel):
    """Response model for predictions"""
    predictions: List[Dict[str, float]] = Field(
        ...,
        description="List of class predictions with confidence scores"
    )
    top_prediction: str = Field(..., description="Most likely class")
    confidence: float = Field(..., description="Confidence of top prediction")
    model_version: str = Field(..., description="Model version used")
    timestamp: datetime = Field(default_factory=datetime.utcnow)

class HealthResponse(BaseModel):
    """Health check response"""
    status: str
    model_loaded: bool
    version: str

class ModelInfo(BaseModel):
    """Model information response"""
    model_name: str
    version: str
    input_shape: List[int]
    num_classes: int
    classes: List[str]
```

### Step 4: Configuration (config.py)
```python
from pathlib import Path

class Settings:
    # API Settings
    API_TITLE = "Image Classification API"
    API_VERSION = "1.0.0"
    API_DESCRIPTION = "REST API for image classification using deep learning"

    # Model Settings
    MODEL_PATH = Path("models/weights/resnet50.pth")
    MODEL_NAME = "ResNet50"
    INPUT_SIZE = (224, 224)
    MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

    # ImageNet classes (top 5 for brevity)
    CLASSES = [
        "tench", "goldfish", "great_white_shark", "tiger_shark", "hammerhead"
        # ... add all 1000 ImageNet classes
    ]

settings = Settings()
```

### Step 5: Utility Functions (utils.py)
```python
import torch
import torchvision.transforms as transforms
from PIL import Image
import io
from typing import Dict, List

# Image preprocessing pipeline
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(
        mean=[0.485, 0.456, 0.406],
        std=[0.229, 0.224, 0.225]
    )
])

def validate_image(file_content: bytes) -> bool:
    """Validate image file"""
    try:
        image = Image.open(io.BytesIO(file_content))
        image.verify()
        return True
    except Exception:
        return False

def preprocess_image(file_content: bytes) -> torch.Tensor:
    """Preprocess image for model input"""
    image = Image.open(io.BytesIO(file_content)).convert('RGB')
    img_tensor = preprocess(image)
    return img_tensor.unsqueeze(0)

def format_predictions(
    probabilities: torch.Tensor,
    classes: List[str],
    top_k: int = 5
) -> Dict:
    """Format model predictions into response"""
    probs = torch.nn.functional.softmax(probabilities[0], dim=0)
    top_probs, top_indices = torch.topk(probs, top_k)

    predictions = [
        {classes[idx]: float(prob)}
        for prob, idx in zip(top_probs, top_indices)
    ]

    return {
        'predictions': predictions,
        'top_prediction': classes[top_indices[0]],
        'confidence': float(top_probs[0])
    }
```

### Step 6: Main API Application (main.py)
```python
from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import RedirectResponse
import torch
import torchvision.models as models
from typing import Optional

from models import PredictionResponse, HealthResponse, ModelInfo
from config import settings
from utils import validate_image, preprocess_image, format_predictions

# Initialize FastAPI app
app = FastAPI(
    title=settings.API_TITLE,
    version=settings.API_VERSION,
    description=settings.API_DESCRIPTION
)

# Global model variable
model: Optional[torch.nn.Module] = None

@app.on_event("startup")
async def load_model():
    """Load model on startup"""
    global model
    try:
        model = models.resnet50(pretrained=True)
        model.eval()
        print("Model loaded successfully")
    except Exception as e:
        print(f"Error loading model: {e}")
        raise

@app.get("/", include_in_schema=False)
async def root():
    """Redirect to API documentation"""
    return RedirectResponse(url="/docs")

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy" if model is not None else "unhealthy",
        "model_loaded": model is not None,
        "version": settings.API_VERSION
    }

@app.get("/info", response_model=ModelInfo)
async def model_info():
    """Get model information"""
    return {
        "model_name": settings.MODEL_NAME,
        "version": settings.API_VERSION,
        "input_shape": [3, 224, 224],
        "num_classes": len(settings.CLASSES),
        "classes": settings.CLASSES[:10]  # Return first 10 classes
    }

@app.post("/predict", response_model=PredictionResponse)
async def predict(file: UploadFile = File(...)):
    """
    Predict image class

    - **file**: Image file (JPEG, PNG)

    Returns predictions with confidence scores
    """
    # Validate file type
    if file.content_type not in ["image/jpeg", "image/png", "image/jpg"]:
        raise HTTPException(
            status_code=400,
            detail="Invalid file type. Only JPEG and PNG are supported."
        )

    # Read file content
    file_content = await file.read()

    # Validate file size
    if len(file_content) > settings.MAX_FILE_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"File too large. Maximum size is {settings.MAX_FILE_SIZE} bytes."
        )

    # Validate image
    if not validate_image(file_content):
        raise HTTPException(
            status_code=400,
            detail="Invalid or corrupted image file."
        )

    try:
        # Preprocess image
        img_tensor = preprocess_image(file_content)

        # Make prediction
        with torch.no_grad():
            outputs = model(img_tensor)

        # Format response
        result = format_predictions(outputs, settings.CLASSES)
        result['model_version'] = settings.API_VERSION

        return result

    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Error during prediction: {str(e)}"
        )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 7: Write Tests (test_api.py)
```python
import pytest
from fastapi.testclient import TestClient
from io import BytesIO
from PIL import Image
import sys
sys.path.insert(0, '.')

from main import app

client = TestClient(app)

def create_test_image(format="JPEG"):
    """Create a test image"""
    img = Image.new('RGB', (224, 224), color='red')
    img_byte_arr = BytesIO()
    img.save(img_byte_arr, format=format)
    img_byte_arr.seek(0)
    return img_byte_arr

def test_health_endpoint():
    """Test health check endpoint"""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert "status" in data
    assert "model_loaded" in data

def test_info_endpoint():
    """Test model info endpoint"""
    response = client.get("/info")
    assert response.status_code == 200
    data = response.json()
    assert "model_name" in data
    assert "num_classes" in data

def test_predict_valid_image():
    """Test prediction with valid image"""
    test_image = create_test_image()
    files = {"file": ("test.jpg", test_image, "image/jpeg")}
    response = client.post("/predict", files=files)

    assert response.status_code == 200
    data = response.json()
    assert "predictions" in data
    assert "top_prediction" in data
    assert "confidence" in data
    assert len(data["predictions"]) == 5

def test_predict_invalid_file_type():
    """Test prediction with invalid file type"""
    files = {"file": ("test.txt", BytesIO(b"not an image"), "text/plain")}
    response = client.post("/predict", files=files)
    assert response.status_code == 400

def test_predict_png_image():
    """Test prediction with PNG image"""
    test_image = create_test_image(format="PNG")
    files = {"file": ("test.png", test_image, "image/png")}
    response = client.post("/predict", files=files)
    assert response.status_code == 200
```

### Step 8: Run the API
```bash
# Start the server
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Run tests
pytest test_api.py -v
```

### Step 9: Test with cURL
```bash
# Health check
curl http://localhost:8000/health

# Get model info
curl http://localhost:8000/info

# Make prediction
curl -X POST "http://localhost:8000/predict" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@image.jpg"
```

## Expected Outputs

### 1. API Documentation
- Automatic Swagger UI at `http://localhost:8000/docs`
- ReDoc documentation at `http://localhost:8000/redoc`
- Interactive API testing interface

### 2. Prediction Response Example
```json
{
  "predictions": [
    {"golden_retriever": 0.87},
    {"labrador": 0.09},
    {"beagle": 0.02},
    {"poodle": 0.01},
    {"terrier": 0.01}
  ],
  "top_prediction": "golden_retriever",
  "confidence": 0.87,
  "model_version": "1.0.0",
  "timestamp": "2025-11-14T10:30:00.123456"
}
```

### 3. Health Check Response
```json
{
  "status": "healthy",
  "model_loaded": true,
  "version": "1.0.0"
}
```

## Bonus Challenges

- [ ] Add support for batch predictions (multiple images)
- [ ] Implement caching for repeated predictions
- [ ] Add image URL support (not just file uploads)
- [ ] Implement model versioning (serve multiple model versions)
- [ ] Add Prometheus metrics endpoint
- [ ] Dockerize the application
- [ ] Add request logging with structured logs (JSON)
- [ ] Implement request ID tracing
- [ ] Add CORS middleware for frontend integration
- [ ] Deploy to cloud platform (AWS, GCP, Heroku)

## Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Models](https://docs.pydantic.dev/)
- [PyTorch Vision Models](https://pytorch.org/vision/stable/models.html)
- [REST API Best Practices](https://restfulapi.net/)
- [Testing FastAPI](https://fastapi.tiangolo.com/tutorial/testing/)

## Success Criteria

- [ ] API starts without errors
- [ ] All endpoints return correct status codes
- [ ] Image upload and prediction works correctly
- [ ] Invalid inputs return appropriate error messages
- [ ] All tests pass
- [ ] API documentation is accessible and accurate
- [ ] Response times are < 1 second for single predictions
- [ ] Code follows PEP 8 style guidelines
